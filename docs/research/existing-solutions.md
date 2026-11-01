# Existing solutions and prior art

Four projects are relevant. Two are official upstream contributions, two are third-party
projects solving adjacent problems. None of them solves *endpoint failover for a single peer
identity*, which is the gap this project fills.

## Comparison

| Project | Problem solved | Changes peer identity? | Uses official APIs? | License | Relevance |
|---|---|---|---|---|---|
| `contrib/reresolve-dns` (upstream) | Endpoint moves because DNS changed | No | Yes (`wg set`) | GPL-2.0 | Direct ancestor — the baseline behaviour |
| `contrib/nat-hole-punching` (upstream) | Two NATed peers find each other | No | Yes (`wg set`) | GPL-2.0 | Demonstrates endpoint mutation as normal operation |
| wgrtc | NAT traversal + endpoint discovery via broker | No | Yes | Apache-2.0 | Closest in spirit; different mechanism |
| PrivyCS | Managing/rotating many complete VPN configs | **Yes** | Partly | GPL-3.0 | Adjacent, explicitly out of our scope |

---

## 1. `contrib/reresolve-dns` — the official baseline

A ~40-line shell script run from cron.

```bash
[[ $(wg show "$INTERFACE" latest-handshakes) =~ ${PUBLIC_KEY//+/\\+}\	([0-9]+) ]] || return 0
(( ($EPOCHSECONDS - ${BASH_REMATCH[1]}) > 135 )) || return 0
wg set "$INTERFACE" peer "$PUBLIC_KEY" endpoint "$ENDPOINT"
```

Source: [`contrib/reresolve-dns/reresolve-dns.sh`](https://github.com/WireGuard/wireguard-tools/blob/master/contrib/reresolve-dns/reresolve-dns.sh).

### What it teaches us

- The sanctioned mutation is `wg set <if> peer <key> endpoint <addr>` — exactly what this
  project uses.
- Upstream's own heuristic for "probably stale" is **135 seconds** since last handshake,
  which sits between `RekeyAttemptTime` (90 s) and `RejectAfterTime` (180 s). It is a
  heuristic, not a protocol guarantee.
- The operation is stateless and idempotent: re-applying the same endpoint is harmless. Our
  engine should be equally safe to run repeatedly.

### What it does not do

- One endpoint per peer. No alternatives.
- No health evaluation — it re-resolves on a timer regardless of whether anything is wrong.
- No failback concept because there is nothing to fail back *to*.
- No hysteresis: if the DNS answer flaps, the endpoint flaps.

**Adoption decision.** Reimplement the semantics, do not copy the code (`wireguard-tools` is
GPL-2.0; this project is Apache-2.0). The 135 s figure informs, but does not determine, our
defaults.

## 2. `contrib/nat-hole-punching` — endpoint mutation as a first-class operation

Two C programs that spoof UDP source ports so two NATed peers can open state and then let a
normal WireGuard handshake complete.

Source: [`contrib/nat-hole-punching`](https://github.com/WireGuard/wireguard-tools/tree/master/contrib/nat-hole-punching).

**Relevance:** it confirms that "change the endpoint of a live peer" is expected, ordinary
operation, and that WireGuard's design deliberately leaves endpoint selection to external
software. It also shows the ceiling of what is possible without touching the protocol: you
can influence *where packets go*, never *what they contain*.

**Non-relevance:** NAT traversal is explicitly out of scope. We assume candidate endpoints
are reachable addresses, not addresses that must be discovered.

## 3. wgrtc — closest third-party project

`gutschke/wgrtc` (Apache-2.0, revision `2bccde1ca6a8`).

> WireGuard tunnels that just work, even when both ends are behind NAT. […] We don't fork
> WireGuard, we don't replace it, we don't re-invent NAT traversal. We glue together pieces
> that already exist — WireGuard for the data plane, a thin WebRTC-style signaling layer for
> endpoint discovery, raw-UDP hole punching for NAT — and we package them so the result is a
> Debian `.deb` on the Linux side and a signed `.apk` on the Android side.

Source: [`gutschke/wgrtc` README](https://github.com/gutschke/wgrtc).

**Architecture:** a Python 3 asyncio daemon that "interrogates `wg show`, exchanges endpoint
envelopes through a signaling broker, and punches NATs open with raw UDP", plus a Kotlin
Android app that is "a WireGuard tunnel manager" which can host a tunnel itself with "full
userspace WireGuard".

### Overlap with this project

- Same core insight: the endpoint is mutable state to be managed externally, and the correct
  primitive is an endpoint write on a live peer.
- Same packaging shape: Debian package on Linux, signed APK on Android.
- Same finding on Android: the app must own the `VpnService` and the userspace tunnel.
- Explicitly handles endpoint re-selection on Wi-Fi ↔ cellular transitions ("every tunnel
  re-races its endpoint candidates independently").

### Differences

- wgrtc *discovers* endpoints through a broker and STUN; it does not take a user-supplied
  ordered list with priorities.
- It depends on a signaling broker, which is exactly the "central relay" this project's
  requirements exclude.
- It is written in Python on Linux and Kotlin on Android — two implementations, which is the
  Option B cost we are trying to avoid.

**What to borrow:** the framing of endpoints as candidates that must be re-raced after a
network change, and the honest Android packaging story.

**What not to borrow:** broker dependency, and the two-language split.

## 4. PrivyCS — adjacent, and deliberately out of scope

`hoep/privycs-vpn` (GPL-3.0, revision `e592c46c4704`), described as "Privycs VPN Client —
Multi-Protocol VPN for Android and iOS".

> **Privycs is a VPN management client, not a VPN service.** […] multi-protocol switching,
> connection pools with rotation, per-pool split tunnel, a real kill switch […]
>
> **Treat 600 configs as one thing.** The Connection Pool feature collapses a whole archive
> into a single virtual connection with three rotation policies (Geo-Nearest, Round-Robin,
> Random), pre-warm probes and recovery against dead servers.

Source: [`hoep/privycs-vpn` README](https://github.com/hoep/privycs-vpn).

**Why it is not the same problem:** a connection pool rotates between *complete
configurations* — different servers, different public keys, different tunnel addresses,
different `AllowedIPs`. Rotating one of those is a teardown and a rebuild of the tunnel, and
the peer identity changes. That is explicitly excluded from this project's MVP.

### What is still worth learning from it

- The user-facing vocabulary — *pool*, *rotation*, *pre-warm probe*, *recovery* — is the
  vocabulary users already have. Our configuration and status output should be legible to
  someone who has used a connection-pool client.
- "Pre-warm probes" is the same problem as our failback attempt: deciding whether a
  candidate is usable before committing. Our answer is a probationary switch with rollback
  (see [`../architecture/health-monitoring.md`](../architecture/health-monitoring.md)),
  because we cannot probe a WireGuard endpoint without committing to it.
- It demonstrates the Android single-`VpnService` constraint in practice: four protocols, one
  slot.

**Licensing note:** GPL-3.0. No code, configuration format, or textual content from PrivyCS
may be incorporated into this Apache-2.0 project.

---

## The gap

Nobody implements the specific combination this project targets:

1. **One peer identity** — public key, preshared key, `AllowedIPs`, tunnel address all fixed.
2. **N ordered candidates** — supplied by the operator, not discovered.
3. **Active health evaluation** — proven failure, not assumed failure, and no false positives
   on an idle tunnel.
4. **Deterministic switching policy** — priority, hysteresis, cooldown, backoff, failback.
5. **No broker, no relay, no central service.**

`reresolve-dns` covers (1) and part of (4) with no candidates. wgrtc covers candidate
re-selection but requires a broker and discovers rather than prioritises. PrivyCS covers
rotation but changes identity.

The design that follows is in [`../architecture/overview.md`](../architecture/overview.md).
