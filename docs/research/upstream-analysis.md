# Upstream analysis: what WireGuard actually supports

This document records what the official WireGuard implementations expose for changing a
peer's endpoint, and what they do not expose. Every claim is traced to source or official
documentation.

## Evidence status

All source references were read at these revisions on 2026-11-01. Line numbers are from
those revisions and may drift.

| Repository | Revision | License |
|---|---|---|
| `WireGuard/wireguard-tools` | `a99840774700` | GPL-2.0 |
| `WireGuard/wireguard-go` | `b3bc704ef0ee` | MIT |
| `WireGuard/wireguard-windows` | `6ece77bc487c` | MIT |
| `WireGuard/wireguard-android` | `e7b3a3c11883` | Apache-2.0 |
| `WireGuard/wgctrl-go` | `a9ab2273dd10` | MIT |
| `WireGuard/wireguard-linux` | `2a73c9b2c0ca` | GPL-2.0 (kernel) |

Statements marked **[unverified]** were not confirmed by reading source or documentation
and must be validated by an experiment before being relied on.

---

## 1. A peer has exactly one endpoint

There is no list type anywhere in the data model. The endpoint is a single attribute.

**Generic netlink UAPI** — `include/uapi/linux/wireguard.h` defines exactly one endpoint
attribute for a peer and no container type around it:

```c
enum wgpeer_attribute {
	WGPEER_A_UNSPEC,
	WGPEER_A_PUBLIC_KEY,
	WGPEER_A_PRESHARED_KEY,
	WGPEER_A_FLAGS,
	WGPEER_A_ENDPOINT,          /* singular */
	WGPEER_A_PERSISTENT_KEEPALIVE_INTERVAL,
	WGPEER_A_LAST_HANDSHAKE_TIME,
	WGPEER_A_RX_BYTES,
	WGPEER_A_TX_BYTES,
	WGPEER_A_ALLOWEDIPS,
	WGPEER_A_PROTOCOL_VERSION,
	...
};
```

Source: [`include/uapi/linux/wireguard.h`](https://github.com/torvalds/linux/blob/master/include/uapi/linux/wireguard.h)
(mirrored at [`WireGuard/wireguard-linux`](https://github.com/WireGuard/wireguard-linux)).

**`wg(8)` CLI** — `set` accepts one endpoint per peer:

```text
wg set <interface> [listen-port <port>] ... [peer <base64-public-key>
   [remove] [preshared-key <file-path>] [endpoint <ip>:<port>] ...
```

Source: [`src/man/wg.8`](https://github.com/WireGuard/wireguard-tools/blob/master/src/man/wg.8).

**`wgctrl-go`** — `wgtypes.PeerConfig.Endpoint` is `*net.UDPAddr`, singular:

```go
// Endpoint specifies the endpoint of this peer entry, if not nil.
Endpoint *net.UDPAddr
```

Source: [`wgtypes/types.go`](https://github.com/WireGuard/wgctrl-go/blob/master/wgtypes/types.go).

**wireguard-go UAPI** — the key/value protocol has a single `endpoint` key:

```go
case "endpoint":
    device.log.Verbosef("%v - UAPI: Updating endpoint", peer.Peer)
    endpoint, err := device.net.bind.ParseEndpoint(value)
    ...
    peer.endpoint.val = endpoint
```

Source: [`device/uapi.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/uapi.go).

**Conclusion.** Multiple endpoints per peer is a data-model change, not a configuration
change. Any solution that does not modify WireGuard must therefore keep the list of
candidates *outside* WireGuard and write one endpoint at a time. This is the foundational
constraint of this project.

## 2. There is no upstream multiple-endpoint feature

A targeted search of the WireGuard organisation's issues and pull requests for the phrase
"multiple endpoints" returns a single unrelated historical hit
(`WireGuard/wireguard-monolithic-historical#1`, "wg-quick: add support for DNS service
lookups"). No open or merged work implements endpoint lists.

**[unverified]** The maintainer's stated position on the mailing list was not located in a
citable form during this research. Do not quote a position that has not been read.

The official answer to "my endpoint changes" is DNS plus periodic re-resolution, shipped as
a contributed script — see §7.

## 3. Endpoints are always stored as resolved `ip:port`

WireGuard never stores a hostname. Resolution happens before the value reaches the
implementation:

- `wg(8)` resolves the hostname and passes a `sockaddr` over netlink
  (`WGPEER_A_ENDPOINT` is a binary address, not a string).
- `wgctrl-go` requires `*net.UDPAddr`, i.e. an already-resolved address.
- wireguard-go resolves inside the bind when it receives a UAPI string:

```go
func (*StdNetBind) ParseEndpoint(s string) (Endpoint, error) {
    ...
    uaddr, err := net.ResolveUDPAddr(...)
```

Source: [`conn/bind_std.go`](https://github.com/WireGuard/wireguard-go/blob/master/conn/bind_std.go).

**Consequence.** DNS is entirely the caller's problem. A companion daemon must resolve
candidate hostnames itself, decide between multiple A/AAAA records, honour TTLs or a
re-resolution interval, and re-apply the endpoint when the answer changes. This is exactly
what `contrib/reresolve-dns` does.

**Consequence for IPv6.** Because the stored value is a sockaddr with a family, an
IPv4 endpoint and an IPv6 endpoint are distinct candidates. Switching between them is a
plain endpoint write; no interface reconfiguration is required. **[unverified]** Whether a
dual-stack peer benefits from preferring one family requires measurement, not assumption.

## 4. Updating an endpoint is peer-scoped and non-destructive

This is the single most important property for the project: **writing an endpoint does not
disturb any other peer attribute.**

The netlink API is a partial-update API. `wgctrl-go` encodes only non-nil fields:

```go
// Because the zero value of some Go types may be significant to WireGuard for
// Config fields, only fields which are not nil will be applied when
// configuring a device.
func (c *Client) ConfigureDevice(name string, cfg wgtypes.Config) error
```

Source: [`client.go`](https://github.com/WireGuard/wgctrl-go/blob/master/client.go).

and in the Linux encoder:

```go
if cfg.Peers[i].Endpoint != nil {
    peer.Flags |= ioctl.PeerHasEndpoint
    ...
}
```

A peer can additionally be marked update-only, so the operation fails rather than silently
creating a peer:

```c
enum wgpeer_flag {
	WGPEER_F_REMOVE_ME = 1,
	WGPEER_F_REPLACE_ALLOWEDIPS = 2,
	WGPEER_F_UPDATE_ONLY = 4,
};
```

`wgctrl-go` maps this to `PeerConfig.UpdateOnly` and emits `WGPEER_F_UPDATE_ONLY`
(`internal/wglinux/configure_linux.go`, `internal/wgwindows/internal/ioctl/configuration_windows.go`).

The minimal, safe endpoint write is therefore:

```go
wgtypes.Config{Peers: []wgtypes.PeerConfig{{
    PublicKey:  peer,
    UpdateOnly: true,       // never create a peer by accident
    Endpoint:   &addr,      // the only field being changed
}}}
```

No key, preshared key, `AllowedIPs`, keepalive or interface address is present in the
message, so none can be altered. This is the mechanism the whole project rests on, and it
is what the test suite must assert byte-for-byte.

**No tunnel restart is involved.** The device stays up, the interface address is untouched,
routes are untouched, and in-flight sessions are not torn down. Existing transport sessions
survive; the next packet is sent to the new endpoint.

## 5. Roaming silently overwrites a manually-set endpoint

WireGuard updates a peer's endpoint from the source address of any correctly authenticated
packet. This is documented behaviour, not a side effect:

> `Endpoint` — an endpoint IP or hostname, followed by a colon, and then a port number.
> This endpoint will be updated automatically to the most recent source IP address and port
> of correctly authenticated packets from the peer.

Source: [`src/man/wg.8`](https://github.com/WireGuard/wireguard-tools/blob/master/src/man/wg.8).

`wgctrl-go` restates it in the read path:

```go
// Endpoint is the most recent source address used for communication by
// this Peer.
Endpoint *net.UDPAddr
```

and wireguard-go implements it:

```go
func (peer *Peer) SetEndpointFromPacket(endpoint conn.Endpoint) {
	peer.endpoint.Lock()
	defer peer.endpoint.Unlock()
	if peer.endpoint.disableRoaming {
		return
	}
	peer.endpoint.clearSrcOnTx = false
	peer.endpoint.val = endpoint
}
```

Source: [`device/peer.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/peer.go).

### Why this matters for failover

There are two sources of truth for "the endpoint": what the manager *wants* and what
WireGuard *currently has*. They can diverge at any time, without our code running.

Design consequences, all of which appear in
[`../architecture/endpoint-manager.md`](../architecture/endpoint-manager.md):

1. The manager must track `desired_endpoint` separately from the observed endpoint, and must
   never assume its last write is still in effect.
2. An observed endpoint that differs from the desired one is **evidence**, not an error:
   - observed address is one of our candidates → the peer roamed; adopt it as active rather
     than fighting it;
   - observed address is unknown → record it, keep policy, re-assert after a cooldown.
3. A switch can be undone by the peer itself a moment later. Health evaluation must read
   state *after* the write and must tolerate this.

This is the difference between a naive "set endpoint and declare victory" implementation and
one that actually works.

## 6. An idle tunnel is not a broken tunnel

> By default, WireGuard tries to be as silent as possible when not being used; it is not a
> chatty protocol. For the most part, it only transmits data when a peer wishes to send
> packets. When it's not being asked to send packets, it stops sending packets until it is
> asked again.

Source: [WireGuard Quick Start — NAT and Firewall Traversal Persistence](https://www.wireguard.com/quickstart/).

Protocol timers, from `wireguard-go/device/constants.go`:

| Constant | Value | Meaning |
|---|---|---|
| `RekeyAfterTime` | 120 s | Rekey once this long has passed, *when there is traffic to send* |
| `RekeyAttemptTime` | 90 s | Give up rekeying after this long |
| `RekeyTimeout` | 5 s | Retry interval for handshake attempts |
| `RejectAfterTime` | 180 s | Transport keys are no longer usable after this |
| `KeepaliveTimeout` | 10 s | Keepalive timing |
| `MaxTimerHandshakes` | 18 | `RekeyAttemptTime / RekeyTimeout` |

Source: [`device/constants.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/constants.go).

**Therefore:**

- `latest-handshake` does **not** advance on a healthy but idle tunnel. A tunnel that has
  been quiet for an hour is normal, not failed. Treating "old handshake" as "endpoint down"
  is the primary design error this project must avoid.
- Conversely, `RejectAfterTime = 180 s` is a hard upper bound on how long a stale session
  can hide a broken path: after 180 s without a successful handshake, the peer cannot send
  or receive usable transport data. This gives a principled maximum detection latency for
  any scheme that relies on handshake observation alone.
- The official `reresolve-dns` script uses a **135 s** threshold — between
  `RekeyAttemptTime` (90 s) and `RejectAfterTime` (180 s) — before re-asserting an endpoint.
  That is a hand-tuned heuristic, not a protocol constant.

Health-check design derived from this is in
[`../architecture/health-monitoring.md`](../architecture/health-monitoring.md).

## 7. The official prior art: `contrib/reresolve-dns`

The only endpoint-maintenance tool shipped by upstream is a shell script.

```bash
[[ $(wg show "$INTERFACE" latest-handshakes) =~ ${PUBLIC_KEY//+/\\+}\	([0-9]+) ]] || return 0
(( ($EPOCHSECONDS - ${BASH_REMATCH[1]}) > 135 )) || return 0
wg set "$INTERFACE" peer "$PUBLIC_KEY" endpoint "$ENDPOINT"
```

Source: [`contrib/reresolve-dns/reresolve-dns.sh`](https://github.com/WireGuard/wireguard-tools/blob/master/contrib/reresolve-dns/reresolve-dns.sh).

Its README:

> Run this script from cron every thirty seconds or so, and it will ensure that if, when
> using a dynamic DNS service, the DNS entry for a hosts changes, the kernel will get the
> update to the DNS entry.

What this establishes, authoritatively:

- `wg set <if> peer <key> endpoint <addr>` **is** the sanctioned way to move a live tunnel.
- Endpoint maintenance is expected to be an external, periodic, stateless operation.
- Upstream considers the problem solved at the "re-resolve and re-apply" level, with no
  notion of candidates, priorities, health or failback.

What it deliberately does not do — and what this project adds — is everything after that:
multiple candidates, active health evaluation, switching policy, hysteresis, and recovery.

Note: `wireguard-tools` is **GPL-2.0**. The script's behaviour may be reimplemented from the
documented semantics; its code must not be copied into this Apache-2.0 project.

## 8. Android deliberately disables roaming

This is an upstream behaviour that materially changes the Android design and is easy to
miss.

`wireguard-android`'s JNI bridge calls:

```go
device.DisableSomeRoamingForBrokenMobileSemantics()
```

Source: [`tunnel/tools/libwg-go/api-android.go`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/tools/libwg-go/api-android.go).

which sets `device.net.brokenRoaming`, and the UAPI handler then disables roaming for a
peer that is created with an endpoint:

```go
if peer.created {
    peer.endpoint.disableRoaming = peer.device.net.brokenRoaming && peer.endpoint.val != nil
}
```

Source: [`device/uapi.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/uapi.go).

**Consequence.** On Android, a peer configured with an endpoint will not follow the peer if
it roams — by design, because mobile networks produce spurious source addresses. For this
project that is mostly convenient (our chosen endpoint stays chosen), but it is a real
behavioural divergence from Linux and Windows and must be documented for users and covered
by platform-specific tests.

Note also that the flag is applied at peer *creation*. Updating the endpoint of an existing
peer does not change `disableRoaming`, so a failover write does not toggle roaming on or off.

## 9. What upstream gives us, summarised

| Capability | Linux | Windows (WireGuardNT) | Android (wireguard-android) |
|---|---|---|---|
| Read peer state | netlink `WG_CMD_GET_DEVICE` | `DeviceIoControl(IOCTL_GET)` | UAPI `get=1` via `wgGetConfig` |
| Write one peer's endpoint | netlink `WG_CMD_SET_DEVICE` | `DeviceIoControl(IOCTL_SET)` | UAPI `set=1` — **not exposed by the app** |
| Update without restart | yes | yes | possible in the engine, not exposed |
| Peer identity preserved | yes (partial update) | yes (partial update) | yes (partial update) |
| Roaming overwrites endpoint | yes | **[unverified]** yes | **disabled** by `brokenRoaming` |
| Official tooling | `wg(8)`, `reresolve-dns` | none for runtime endpoint | none |

Per-platform detail, privileges and gaps:
[`platform-capabilities.md`](platform-capabilities.md).

## 10. Open questions carried forward

1. Does WireGuardNT's driver accept an endpoint-only `IOCTL_SET` on an adapter that is
   already `Up`, while traffic flows? Source says the flags exist; behaviour must be proven
   in milestone 4.
2. What is the observable effect on an in-flight transport session when the endpoint changes
   mid-stream — dropped packets, retransmission, or nothing? Measurable in milestone 3.
3. Does the Linux kernel module rate-limit handshake initiations per peer in a way that
   interferes with aggressive health checks? Check `CookieRefreshTime`/`HandshakeInitationRate`
   behaviour empirically.
4. On Android, is there any supported path to change an endpoint without patching
   `tunnel`? Source review says no; confirm against the current release artifact.
