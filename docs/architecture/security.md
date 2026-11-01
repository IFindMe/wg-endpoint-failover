# Security model

## 1. What this component is, in security terms

`wg-endpoint-failover` is a **control-plane** component. It decides where WireGuard sends
packets. It never touches the data plane: it does not encrypt, decrypt, encapsulate, or
inspect tunnel payloads.

That gives it an unusual risk profile:

- A compromise **does not** yield plaintext, session keys, or the ability to impersonate the
  peer.
- A compromise **does** yield the ability to redirect traffic to an address of the attacker's
  choosing, which is a denial of service and a metadata-collection primitive if the attacker
  controls the destination network path.

Stated plainly: **this is not a key-management component, and it must never become one.**

## 2. Assets and classification

| Asset | Classification | Handled? | Where it lives |
|---|---|---|---|
| WireGuard private key | Critical secret | **Never** | WireGuard's own configuration only |
| Preshared key | Critical secret | **Never** | WireGuard's own configuration only |
| Peer public keys | Public | Yes | Companion config, logs, status |
| Endpoint addresses | Sensitive metadata | Yes | Companion config, logs, status |
| Tunnel plaintext | User data | **Never** | Nowhere in this component |
| Companion config | Integrity-critical | Yes | Root-owned file / ProgramData / app sandbox |
| Runtime state | Low sensitivity | Yes | Daemon memory + state file |

## 3. What an attacker can and cannot do

This section exists because the common intuition ("if they control the endpoint they can read
my traffic") is wrong, and the wrong intuition leads to the wrong mitigations.

### 3.1 Cannot: decrypt or impersonate by changing the endpoint

WireGuard authenticates *peers*, not *paths*. Pointing the endpoint at an attacker-controlled
address achieves nothing unless that address holds the peer's private key and can complete a
Noise handshake. If it cannot, the handshake fails and the tunnel is dead.

So endpoint redirection is a **denial-of-service** primitive, not a confidentiality breach.

### 3.2 Can: deny service

An attacker who can cause repeated failover can keep the tunnel down, or move it to an endpoint
that cannot complete a handshake. Mitigations:

- candidates are operator-supplied and validated; the engine never invents one,
- the global switch rate limit stops pathological churn,
- `FLAPPING` is reported rather than silently continuing.

### 3.3 Can: influence which endpoint is chosen, if DNS is compromised

DNS is unauthenticated. A poisoned answer redirects a *hostname* candidate to an attacker
address. Consequences:

- the handshake fails → the candidate is marked failed → the engine moves on (self-correcting),
- or, if the attacker can also complete the handshake, they already hold the peer's private
  key and the game is over regardless of this component.

Recommended mitigation, documented for users rather than enforced: use IP-literal candidates
for the primary, or DNSSEC-validating resolvers, or both. The engine cannot verify DNS and
must not pretend to.

### 3.4 Can: redirect traffic if it can write the companion config

This is the realistic attack. If an unprivileged user can edit `/etc/wg-endpoint-failover/
config.toml`, they can add a candidate and (given enough control over the network path) cause
the tunnel to point elsewhere.

Mitigation: strict file permissions, checked at startup (§4.3). This is the single most
important operational control, which is why the daemon refuses to run with a writable config.

### 3.5 Can: observe metadata

Logs contain endpoints and public keys. On a shared host, that reveals which networks a user
connects to. Mitigation: `redact_endpoints` option; log files created `0600`; documented.

## 4. Privilege model

### 4.1 The minimum privilege per platform

| Platform | What is actually needed | Why |
|---|---|---|
| Linux | `CAP_NET_ADMIN` | Generic netlink `WG_CMD_SET_DEVICE` and `WG_CMD_GET_DEVICE` |
| Windows | Membership in `Administrators` (service runs as `LocalSystem`) | Open the WireGuardNT adapter device and issue configuration IOCTLs |
| Android | Nothing beyond the app sandbox | The tunnel is in-process; no kernel control surface |

None of these requires reading a key file. In particular, on Linux `CAP_NET_ADMIN` alone is
sufficient — the daemon does **not** need `CAP_DAC_OVERRIDE` to read `/etc/wireguard/wg0.conf`,
because it never reads that file.

### 4.2 Linux: systemd hardening

```ini
[Service]
Type=notify
ExecStart=/usr/bin/wg-endpoint-failover --config /etc/wg-endpoint-failover/config.toml
Restart=on-failure
RestartSec=5s

# The only capability required.
AmbientCapabilities=CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_ADMIN
NoNewPrivileges=yes
User=root

# Filesystem: the daemon needs its own config and nothing else.
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ReadOnlyPaths=/etc/wg-endpoint-failover
ReadWritePaths=/var/lib/wg-endpoint-failover

# Kernel and namespace surface.
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM

# Networking: it needs to talk to the resolver and the kernel, nothing else.
RestrictAddressFamilies=AF_UNIX AF_NETLINK AF_INET AF_INET6
```

`User=root` is retained because `CAP_NET_ADMIN` must be in the effective set and the daemon
must not depend on ambient capability inheritance quirks. A dedicated user with
`AmbientCapabilities` is preferred and should be validated in milestone 2; the file above is
the conservative baseline.

Note that `ProtectSystem=strict` plus `ReadOnlyPaths=/etc/wireguard` means the daemon cannot
write a WireGuard configuration even if a future bug tried to.

### 4.3 Linux: config file integrity check

At startup, `stat` the configuration file and refuse to start unless:

- owner is `root` (uid 0), and
- mode has no group-write and no other-write bits (`mode & 0022 == 0`).

Error message must be actionable:

```text
config: /etc/wg-endpoint-failover/config.toml is group- or world-writable (mode 0664).
Refusing to start: this file controls where the tunnel sends traffic.
Fix with: chown root:root /etc/wg-endpoint-failover/config.toml && chmod 0640 /etc/wg-endpoint-failover/config.toml
```

### 4.4 Windows: service separation

Mirror the architecture WireGuard for Windows already uses:

```text
wgefctl.exe        (unprivileged CLI)  --named pipe-->  wg-endpoint-failover.exe (service, LocalSystem)
```

- The service holds the adapter handle and performs all writes.
- The CLI is read-only: it queries status over the pipe. It cannot request an endpoint change.
- The named pipe is created with a security descriptor allowing only `Administrators` and
  `LocalSystem`, following the convention `\\.\pipe\ProtectedPrefix\Administrators\...`.
- The service has no UI and no network listeners.

Deliberately **not** done: exposing a "set endpoint" RPC to unprivileged callers. The CLI does
not need it, and every additional caller is an additional path to redirect the tunnel.

### 4.5 Android: no extra privilege

The engine runs inside the app that already owns the `VpnService`. It needs no new permission.
The relevant control is that the app's own configuration is in private storage, so no other app
can add a candidate.

### 4.6 Peer attribute write surface

Exactly two peer attributes may ever be written, and the set is closed by review and by test:

| Attribute | When | Guard |
|---|---|---|
| `endpoint` | Always, on every switch | The whole purpose of the component |
| `persistent_keepalive_interval` | Only in `keepalive` probe mode, when `probe.allow_keepalive = true` | Original value recorded, change logged with before/after, restored on shutdown |

Never written: `private_key`, `preshared_key`, `allowed_ips` (including
`replace_allowed_ips`), `remove`, `protocol_version`, or any interface attribute
(`listen_port`, `fwmark`, interface addresses).

The enforcement is the adapter interface itself
([overview §6](overview.md#6-adapter-interface)): `SetEndpoint` accepts only an address, and
there is no second write method. The keepalive override is a deliberate, narrower exception
implemented through the same interface, so it is subject to the same single-writer path, the
same cooldown, and the same logging.

A test asserts that the set of peer attributes the engine can produce is a subset of this
table. If a future change adds a third attribute, that test fails until this section and the
test are updated together.

## 5. Secret handling rules

These are enforced by construction, then by test.

| Rule | Enforcement |
|---|---|
| Never read a private or preshared key | The adapter interface has no method that could return one (`PeerState` has no key field) |
| Never accept key material in the companion config | Validator rejects `private_key`, `preshared_key`, `psk` anywhere (§ configuration 5) |
| Never log key material | Log helper rejects fields named `*key*` unless they are `public_key`; redaction is default-on for `preshared`/`private` |
| Never shell out to `wg showconf` | No subprocess execution in the daemon at all |
| Never dump full device config | `status` is a projection with an explicit allowlist of fields, not a serialisation of `wgtypes.Device` |

The last rule is subtle and important: `wgtypes.Device` **contains** `PrivateKey` and
`Peer.PresharedKey` fields. Serialising it directly — the obvious implementation — would leak
both. The status type is therefore a separate struct, and there must be a test that fails if
any field of `wgtypes.Device` is ever embedded in a status or log structure.

## 6. Configuration input validation

All input is untrusted. The validator must:

- reject unknown keys (typos that silently disable failover are a safety problem),
- bound every numeric field (a `check_interval` of `1ns` would be a local DoS),
- reject interface names that do not match WireGuard's grammar,
- reject addresses that are not `host:port`,
- reject `priority` collisions,
- refuse `probe_timeout >= check_interval`,
- refuse any document containing key material.

Handling of malformed DNS responses and unavailable resolvers is specified in
[health-monitoring.md §6.5](health-monitoring.md#65-dns-re-resolution): they produce
`UNRESOLVED`, never a crash, never a failure of the endpoint.

## 7. Race and concurrency safety

| Risk | Mitigation |
|---|---|
| Two switches applied concurrently | Single `apply()` path with a mutex; generation counter discards stale decisions ([endpoint-manager §3](endpoint-manager.md#3-switching)) |
| A slow DNS lookup applying an outdated decision | Generation check before write |
| Health check observing a half-applied state | Lock held across write + re-read |
| Two daemon instances on one interface | Exclusive lock file (Linux: `flock` on `/run/wg-endpoint-failover/<iface>.lock`; Windows: named mutex; Android: single process) |
| Shutdown racing an in-flight switch | `Restoring` stops the evaluation loop first, then waits for the writer to drain |

The instance lock is required. Two daemons with different candidate lists would fight, and the
result would be continuous switching — the worst possible failure mode.

## 8. Crash and restart safety

Full analysis in [failure-modes.md](failure-modes.md#3-daemon-crash-and-restart). Security
summary:

- The daemon writes a small state file containing `original_endpoint`, the keepalive override,
  and the daemon's own PID/start time. It contains **no key material and no secrets**.
- On restart, if the state file indicates an unclean exit, the daemon reconciles rather than
  blindly restoring: it reads the live endpoint, and if it is a known candidate it continues
  from there instead of yanking the tunnel.
- The state file is written atomically (temp file + `rename`) with `0600`.

## 9. Supply chain

| Concern | Control |
|---|---|
| Dependencies | Minimal by design: `wgctrl-go` (MIT), a TOML parser, `golang.org/x/sys`, `mdlayher/netlink` (transitive). Each new direct dependency needs an ADR. |
| Upstream trust | `wgctrl-go` is the same library `wg-quick`-adjacent tooling uses; it is pure Go on Linux. On Windows it is source-correct but not CI-exercised upstream — a reason to keep our Windows adapter thin and to test it ourselves. |
| Release integrity | SHA256 checksums for every artifact; Android APK signed with a key held in GitHub Secrets; Windows MSI Authenticode-signed. See [release-plan.md](../release-plan.md). |
| Build provenance | GitHub Actions builds from tagged commits only; no artifact is published from a pull request. |
| Dependency scanning | `govulncheck` (Go) and Dependabot, in CI from milestone 1. |

## 10. Out of scope for this component

Explicitly not this project's job, and should not be attempted here:

- protecting the WireGuard private key at rest (that is WireGuard's and the OS's job),
- authenticating endpoints (WireGuard authenticates peers; the path is not authenticated),
- detecting an on-path attacker who can complete handshakes (impossible without changing the
  protocol),
- protecting tunnel plaintext (never visible here),
- providing a kill switch (that is firewall policy, and each platform already has mechanisms).

## 11. Reporting

See [SECURITY.md](../../SECURITY.md). The distinction between "can redirect the tunnel" and
"can read the tunnel" must be stated in every advisory, because it determines severity.
