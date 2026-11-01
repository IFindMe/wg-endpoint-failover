# Companion configuration

## 1. Principle

> The companion configuration describes **where else the peer might be**. It does not describe
> the peer.

WireGuard's own configuration is the single source of truth for identity: private key, peer
public keys, preshared keys, `AllowedIPs`, listen port, interface addresses. The companion
file never restates any of that, and never contains a secret.

This has a direct consequence for the implementation: **the engine never parses a WireGuard
configuration file.** It reads live peer state from the device through the adapter, which
returns only non-secret fields (`Endpoint`, `LastHandshake`, counters, `AllowedIPs`,
`PersistentKeepalive`). Reading `wg0.conf` would mean reading a private key, so we do not.

## 2. Why a separate file at all

Extending `[Peer]` with a list is not possible without breaking compatibility:

- `wg setconf` / `wg-quick` reject unknown keys — a `Endpoint = a, b` line makes the file
  invalid, not merely ignored.
- `wg-quick strip` passes `[Peer]` sections through to `wg setconf`, so the invalid key
  reaches WireGuard.
- Even if a fork tolerated it, the file would no longer be portable to any other WireGuard
  client.

The companion file is therefore additive and optional. A user who removes it keeps a perfectly
valid, unchanged WireGuard configuration — that is the compatibility guarantee.

## 3. Format

**TOML.** Chosen over the alternatives for concrete reasons:

| Format | Why not |
|---|---|
| YAML | Indentation-sensitive; a mis-indented list silently changes meaning. Large parser surface. |
| JSON | No comments; a configuration file that cannot explain itself. |
| INI (WireGuard style) | No arrays of tables; the endpoint list would need an ad-hoc sub-format, which is exactly what we are avoiding. |
| Extend `.conf` | Breaks compatibility (§2). |

TOML has first-class arrays of tables (`[[peer.endpoint]]`), comments, explicit types, and a
strict decoder in Go (`github.com/pelletier/go-toml/v2` with `DisallowUnknownFields`). Strict
decoding is not a nicety here: it turns a typo like `failover_enabled` into an error instead of
a silently ignored setting that makes failover not work.

## 4. Schema

```toml
schema_version = 1

# ---------------------------------------------------------------- interface
[[interface]]
name = "wg0"                     # must match an existing WireGuard interface
restore_on_shutdown = true       # write the original endpoint back on clean exit

  # ------------------------------------------------------------ failover
  [interface.failover]
  check_interval          = "5s"
  probe_timeout           = "3s"
  failure_threshold       = 3
  recovery_threshold      = 3
  failback                = true
  failback_interval       = "60s"
  min_switch_interval     = "10s"
  candidate_backoff_base  = "30s"
  candidate_backoff_max   = "15m"
  candidate_backoff_factor = 2.0
  backoff_jitter          = 0.2
  max_switches_per_hour   = 12
  network_grace           = "10s"
  roam_reassert_after     = "5m"
  max_inconclusive_cycles = 0      # 0 = derive from the protocol's RejectAfterTime

  # ---------------------------------------------------------------- dns
  [interface.dns]
  refresh_interval = "60s"
  stale_max        = "5m"

  # -------------------------------------------------------------- probe
  [interface.probe]
  mode            = "auto"          # auto | target | keepalive | passive
  targets         = ["10.0.0.1"]    # in-tunnel addresses
  protocol        = "icmp"          # icmp | tcp
  tcp_port        = 443             # required when protocol = "tcp"
  allow_keepalive = true            # permit temporary persistent-keepalive override

  # --------------------------------------------------------------- peer
  [[interface.peer]]
  public_key = "xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg="

    [[interface.peer.endpoint]]
    id       = "primary"
    address  = "endpoint-a.example.com:51820"
    priority = 1
    family   = "auto"               # auto | ipv4 | ipv6

    [[interface.peer.endpoint]]
    id       = "backup-1"
    address  = "173.1.1.40:51820"
    priority = 2

    [[interface.peer.endpoint]]
    id       = "backup-2"
    address  = "backup.example.com:51624"
    priority = 3
```

### Minimal valid configuration

```toml
schema_version = 1

[[interface]]
name = "wg0"

  [[interface.peer]]
  public_key = "xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg="

    [[interface.peer.endpoint]]
    address  = "endpoint-a.example.com:51820"
    priority = 1

    [[interface.peer.endpoint]]
    address  = "173.1.1.40:51820"
    priority = 2
```

Every omitted value takes its default (§6).

## 5. Validation rules

Validation is total: either the whole file is accepted or nothing is applied. There is no
partial configuration.

### Structural

| Rule | On violation |
|---|---|
| `schema_version` present and equal to `1` | error |
| At least one `[[interface]]` | error |
| `interface.name` non-empty | error |
| Unknown keys anywhere | **error** (strict decoding) |
| Duplicate `interface.name` | error |

### Interface name

Must match WireGuard's own interface name grammar, `[a-zA-Z0-9_=+.-]{1,15}`, because that is
what `ip link` and the Windows/Android implementations accept
([`Tunnel.NAME_PATTERN`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/src/main/java/com/wireguard/android/backend/Tunnel.java),
`wg-quick(8)`). Rejecting early gives a better error than failing at the netlink layer.

### Peer

| Rule | On violation |
|---|---|
| `public_key` present | error |
| `public_key` is valid base64 decoding to exactly 32 bytes | error |
| Duplicate `public_key` within one interface | error |
| At least one `endpoint` | error |
| Fewer than two `endpoint` entries | **warning** — valid but nothing to fail over to |

### Endpoint

| Rule | On violation |
|---|---|
| `address` parses as `host:port` with a bracketed IPv6 literal when needed | error |
| port in `1..65535` | error |
| `priority` ≥ 1 | error |
| duplicate `(priority)` within a peer | error (ambiguous ordering) |
| `id` unique within a peer | error |
| `id` absent | defaults to `address` |
| `family` in `{auto, ipv4, ipv6}` | error |
| `family` contradicts a literal address (e.g. `ipv4` with an IPv6 literal) | error |

### Failover parameters

| Rule | On violation |
|---|---|
| durations parse and are > 0 | error |
| `failure_threshold` ≥ 1, `recovery_threshold` ≥ 1 | error |
| `0 ≤ backoff_jitter < 1` | error |
| `candidate_backoff_factor` ≥ 1.0 | error |
| `candidate_backoff_max` ≥ `candidate_backoff_base` | error |
| `failback_interval` ≥ `min_switch_interval` | error |
| `max_switches_per_hour` ≥ 1 | error |
| `probe_timeout` < `check_interval` | error (a probe that outlives its cycle cannot work) |

### Probe

| Rule | On violation |
|---|---|
| `mode` in `{auto, target, keepalive, passive}` | error |
| `mode = "target"` with empty `targets` | error |
| `protocol = "tcp"` with no `tcp_port` | error |
| `mode = "passive"` | **warning**: cannot detect a dead endpoint on an idle tunnel |
| `targets` not covered by the peer's live `AllowedIPs` | **warning**: probe will not enter the tunnel |

The last check is worth highlighting: it requires reading `AllowedIPs` from the **live device**
(not from a config file), which is available without any secret. Catching a misconfigured
probe target at startup prevents a confusing "everything reports INCONCLUSIVE" experience.

### Secrets

The validator rejects the following keys anywhere in the document with an explicit message:

```text
private_key, PrivateKey, preshared_key, PresharedKey, psk
```

This is a guard against the common mistake of pasting a WireGuard config into the companion
file. The error must say *why*, not just *no*:

```text
config: interface[0].peer[0]: field "private_key" is not permitted:
  this file must never contain key material. Endpoints are associated with a peer by its
  public key. See docs/architecture/configuration.md.
```

## 6. Defaults

| Parameter | Default | Rationale |
|---|---|---|
| `restore_on_shutdown` | `true` | Predictable state after stop, as required |
| `check_interval` | `5s` | Balance between detection speed and traffic |
| `probe_timeout` | `3s` | One RTT plus margin on a typical path |
| `failure_threshold` | `3` | 15 s to declare failure in `target` mode |
| `recovery_threshold` | `3` | Prevents promoting an endpoint that answered once |
| `failback` | `true` | "Prefer the primary when healthy" |
| `failback_interval` | `60s` | Fast enough to matter, slow enough not to churn |
| `min_switch_interval` | `10s` | Anti-flap floor |
| `candidate_backoff_base` | `30s` | First penalty |
| `candidate_backoff_max` | `15m` | Ceiling; keeps a dead primary from being retried forever |
| `candidate_backoff_factor` | `2.0` | 30 s → 60 s → 120 s → 240 s → 480 s → 900 s |
| `backoff_jitter` | `0.2` | ±20 %; prevents synchronised retries across hosts |
| `max_switches_per_hour` | `12` | Safety valve against genuine oscillation |
| `network_grace` | `10s` | Absorbs local interface transitions |
| `roam_reassert_after` | `5m` | Long enough not to fight legitimate roaming |
| `max_inconclusive_cycles` | `0` (derived) | `ceil(180s / check_interval)` |
| `dns.refresh_interval` | `60s` | Comparable to `reresolve-dns`'s 30 s cron, less aggressive |
| `dns.stale_max` | `5m` | How long a last-known-good address may be reused |
| `probe.mode` | `auto` | `target` if targets are set, else `keepalive` |
| `probe.protocol` | `icmp` | Cheapest stimulus; the handshake is the real signal |
| `probe.allow_keepalive` | `true` | Required for `keepalive` mode to work |

### Android overrides

The same schema is used on Android, but the shipped defaults differ because battery matters
more than detection speed:

| Parameter | Android default |
|---|---|
| `check_interval` | `30s` |
| `failure_threshold` | `3` |
| `failback_interval` | `300s` |
| `dns.refresh_interval` | `300s` |
| `probe.mode` | `auto` (target preferred) |

Defaults are compiled per platform, not written into the file, so a user who copies a
configuration between platforms gets platform-appropriate behaviour.

## 7. File locations and permissions

| Platform | Path | Owner / mode |
|---|---|---|
| Linux | `/etc/wg-endpoint-failover/config.toml` | `root:root`, `0640` |
| Windows | `%ProgramData%\wg-endpoint-failover\config.toml` | `Administrators`, not writable by users |
| Android | app-private storage | app sandbox |

The file must not be world-writable or group-writable: an attacker who can edit it can redirect
the tunnel (see [security.md §4.3](security.md#43-linux-config-file-integrity-check)). The daemon refuses to
start if the Linux file's mode allows group or other writes.

## 8. Relationship to the WireGuard configuration

```text
/etc/wireguard/wg0.conf                 companion config.toml
-------------------------               ------------------------
[Interface]                             [[interface]]
  PrivateKey = <secret>                   name = "wg0"          <-- reference only
  Address    = 10.0.0.2/24
  ListenPort = 51820                    [[interface.peer]]
                                          public_key = "<same key>"
[Peer]                                    [[interface.peer.endpoint]]
  PublicKey  = <same key>                   address = "a.example.com:51820"
  PresharedKey = <secret>                   priority = 1
  AllowedIPs = 0.0.0.0/0                  [[interface.peer.endpoint]]
  Endpoint   = a.example.com:51820          address = "173.1.1.40:51820"
                                            priority = 2
```

The two files are joined by exactly two values: the interface **name** and the peer **public
key**. Neither is secret. Everything else in the WireGuard file is invisible to this project.

### What happens if they disagree

| Situation | Behaviour |
|---|---|
| Interface in companion config does not exist | `Halted` with a clear error; nothing is written |
| Peer public key not present on the interface | `Halted` with a clear error; nothing is written |
| WireGuard `Endpoint` is not one of the candidates | Accepted; recorded as `original_endpoint`, and a candidate is selected |
| WireGuard `Endpoint` is empty | Accepted; first candidate is written immediately |
| Companion config has no entry for a peer that exists | That peer is untouched — the engine is not a general configuration manager |

The last row is important: the engine only manages peers it was told about. It never enumerates
peers and applies policy to them.

## 9. Trade-offs accepted

| Decision | Benefit | Cost |
|---|---|---|
| Separate file, not extended `.conf` | Full compatibility; no invalid WireGuard files | Two files to keep in sync |
| Peer referenced by public key | No secrets; unambiguous | User must copy a base64 key |
| Strict unknown-key rejection | Typos fail loudly | Slightly hostile to hand-edited configs with comments-as-keys; mitigated by clear errors |
| TOML | Arrays of tables, comments, strict decoder | Less familiar than YAML to some operators |
| Defaults compiled per platform | Sane behaviour everywhere | Documented defaults differ by platform, which can surprise |
| `schema_version` from day one | Future migrations are possible | One more required field |
| Multiple interfaces in the schema from day one | No breaking change later | Marginally more complex validation now |

### Rejected alternatives

| Alternative | Why rejected |
|---|---|
| `Endpoint = a, b, c` in `[Peer]` | Invalid WireGuard configuration; breaks `wg setconf`, `wg-quick`, and every other client |
| Separate `.conf` per candidate | Duplicates peer identity, which the requirements forbid; impossible to keep keys consistent |
| Store the private key to enable probing | Violates the primary security invariant |
| Reference peers by index or by `AllowedIPs` | Ambiguous and unstable across configuration edits |
| Discover candidates automatically | Out of scope; requires a broker or relay |

## 10. Schema versioning

`schema_version` is an integer, currently `1`. Rules:

- Adding an optional field with a default: no version bump.
- Changing a default: no version bump, but documented in the changelog.
- Renaming, removing, or changing the meaning of a field: version bump, with a migration path
  and a deprecation window of at least one minor release.
- The daemon refuses a `schema_version` it does not know, with an error naming the version it
  supports. It does not attempt best-effort parsing.
