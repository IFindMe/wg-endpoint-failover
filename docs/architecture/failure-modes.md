# Failure-mode analysis

Every mode below is either handled by design, detected and surfaced, or explicitly accepted
with a stated residual risk. Nothing is listed as "will not happen".

Severity: **S1** tunnel down or traffic misdirected; **S2** degraded behaviour; **S3**
cosmetic or operational annoyance.

---

## 1. Endpoint and network failures

| ID | Mode | Symptom | Detection | Mitigation | Residual risk |
|---|---|---|---|---|---|
| E1 | Primary endpoint down | Traffic stops; `last_handshake` stops advancing | `FAILURE` after `failure_threshold` cycles once the session expires | Switch to next candidate in priority order; probation confirms | Detection takes up to `RejectAfterTime` (180 s) in `keepalive` mode; immediate in `target` mode |
| E2 | All endpoints down | No candidate passes probation | Every candidate `FAILED` and in backoff | Enter `Backoff`; keep rotating with exponential backoff + jitter; keep the tunnel up; log once per cycle | Tunnel stays dead until the network returns — correct, since tearing it down is worse |
| E3 | Endpoint flaps (up/down repeatedly) | Repeated switches | Switch counter approaches `max_switches_per_hour` | Hysteresis, per-candidate backoff, `min_switch_interval`; on exceeding the limit, stop switching and report `FLAPPING` | A genuinely marginal link still produces some churn before the valve closes |
| E4 | DNS answer changes | Old address stops working | `dns.refresh_interval` re-resolution; or `FAILURE` | Resolve again, switch to the new address as a sub-candidate of the same logical endpoint | Up to one refresh interval of staleness |
| E5 | DNS returns multiple addresses | Only one can be active | — | Addresses are sub-candidates; try next on failure | Ordering within a hostname is a policy guess, not a measurement |
| E6 | DNS unavailable / SERVFAIL / malformed | Hostname candidates unresolvable | Resolution error | Mark `UNRESOLVED`; keep last-known-good for `dns_stale_max`; **do not** count as endpoint failure; IP-literal candidates unaffected | If the outage outlasts `stale_max` and no literal candidates exist, the engine can only report `UNRESOLVED` |
| E7 | NAT rebinding (peer's public port changes) | Traffic stops | Handshake stops completing | Roaming may recover automatically; if not, our endpoint write to the same address re-establishes the path | A peer that changes address without sending authenticated packets is unreachable by definition |
| E8 | IPv6 path broken, IPv4 fine | One family fails | `FAILURE` on the IPv6 candidate | `family = auto` falls back to the other family | No happy-eyeballs racing by design (§ health 6.6) |
| E9 | Endpoint reachable but peer's `AllowedIPs` exclude the probe target | Probes never return | `INCONCLUSIVE` forever, or L2-only `HEALTHY` | Startup warning when targets are not covered by live `AllowedIPs` | Misconfiguration is warned about, not prevented |
| E10 | Peer silently drops probe payload but completes handshakes | L3 never confirms | `handshake_advanced` still true → `HEALTHY` at L2 | L2 is treated as sufficient | Cannot distinguish "peer is up but not forwarding" without a target that replies; documented |
| E11 | ICMP filtered on the whole path | `target` mode with `protocol = icmp` gets no reply | `INCONCLUSIVE`, escalating after `max_inconclusive_cycles` | Configure `protocol = "tcp"` with a port known to respond, or rely on L2 | Detection latency becomes bounded by key expiry |

## 2. Local-environment failures

| ID | Mode | Symptom | Detection | Mitigation | Residual risk |
|---|---|---|---|---|---|
| L1 | WireGuard interface missing | Nothing works | `Read` returns `os.ErrNotExist` | `Halted` with an actionable error; **no endpoint write** | Operator must start the interface |
| L2 | Insufficient privilege | Writes fail with `EPERM`/`EACCES` | Error from `SetEndpoint` | `Halted`; log the required capability; do not retry in a loop | — |
| L3 | Local network change (Wi-Fi ↔ cellular, route change) | All endpoints appear to fail simultaneously | Platform network-change notification | Reset counters, grace period, immediate re-evaluation, immediate DNS refresh | Platform notification latency; Android may coalesce |
| L4 | Interface goes down and up (e.g. `wg-quick down/up`) | Peer state resets; `last_handshake` zeroed | `Read` shows zero handshake and possibly empty endpoint | Treat as `LOCAL_NETWORK_DOWN` for one grace period, then re-evaluate from scratch | The original endpoint recorded at startup may no longer be meaningful; restore is skipped if the interface was recreated |
| L5 | System clock jumps backwards | Handshake age negative; timers misbehave | `last_handshake > now` | Use a monotonic clock for intervals; treat negative age as "just handshaked" | Wall-clock timestamps in status may be confusing after a jump |
| L6 | Second daemon instance on the same interface | Continuous fighting | Exclusive lock acquisition fails | Refuse to start with a clear error | — |
| L7 | Resolver configuration changes (VPN up/down changes `/etc/resolv.conf`) | Resolution starts failing or returns different answers | Resolution error or changed answer | Re-resolve on network change | Transient failures during the change are reported as `UNRESOLVED` |
| L8 | Out of file descriptors / memory pressure | Errors from platform calls | Returned errors | `Halted` after repeated `LOCAL_ERROR`; do not fail over | — |

## 3. Daemon crash and restart

The requirement is "handle daemon crashes and restarts safely". Concretely:

### 3.1 State file

`/var/lib/wg-endpoint-failover/state-<iface>.json` (Linux), `%ProgramData%\...\state-*.json`
(Windows), app-private storage (Android). Written atomically (`temp` + `rename`), mode `0600`.

```json
{
  "schema_version": 1,
  "interface": "wg0",
  "peer": "xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=",
  "original_endpoint": "198.51.100.9:51820",
  "active_candidate_id": "backup-1",
  "keepalive_override": null,
  "pid": 4821,
  "started_at": "2026-11-01T09:00:00Z",
  "clean_shutdown": false
}
```

No key material. No secrets. Contains only what is needed to restore a predictable state.

### 3.2 Restart sequence

```text
1. Read state file, if present.
2. If clean_shutdown == false (previous run crashed):
     a. Read the live endpoint from the device.
     b. If live endpoint is one of our candidates -> adopt it, continue.
        (Do not yank a working tunnel back to the original endpoint.)
     c. If live endpoint is unknown -> treat it as the new original, and select a candidate.
     d. If a keepalive_override was recorded -> re-apply or clear it, per configuration.
3. If clean_shutdown == true:
     a. The original endpoint was already restored; nothing to do.
4. Write a fresh state file with clean_shutdown = false.
```

The important design choice is (b): a crash is not a reason to interrupt a working tunnel.
"Restore a predictable state" means predictable *on clean shutdown*, and reconciling *after a
crash*.

### 3.3 Crash modes and consequences

| Crash point | Consequence | Recovery |
|---|---|---|
| Before any write | Nothing changed | Normal start |
| Between write and state update | Tunnel is on a candidate the state file does not know | Live read wins (§3.2 b) |
| During state file write | Temp file left behind; original intact (atomic rename) | Ignore stale temp file |
| During `Restoring` | Tunnel may be on original or on a candidate | Next start reconciles from live state |
| Repeated crash-loop | systemd `Restart=on-failure` with `RestartSec`; after N failures, `Halted` and alert | Operator intervention |

## 4. Configuration and data-safety failures

| ID | Mode | Symptom | Detection | Mitigation | Residual risk |
|---|---|---|---|---|---|
| C1 | Invalid companion config | Daemon will not start | Validation error | Total validation; actionable message with file/line/field | — |
| C2 | Unknown key (typo) | Failover silently misconfigured | Strict decoding | Hard error, not a warning | — |
| C3 | Key material pasted into companion config | Secret on disk in the wrong place | Validator rejects `private_key`/`preshared_key`/`psk` | Refuse to start with an explanation | The secret is already on disk; operator must rotate it |
| C4 | WireGuard config overwritten or corrupted | User's setup destroyed | — | **Structurally impossible**: the engine never opens a WireGuard config file | — |
| C5 | Unrelated peer settings changed | Surprise for the user | Test suite asserts byte-identical peer state | Partial update sends only `endpoint` (and, opt-in, keepalive) | The keepalive override is a real write; it is recorded, logged, and restored |
| C6 | Config changed while running | Two sources of truth | SIGHUP / file watcher | `Reload` validates fully before applying; failure leaves the old config running | A reload during a switch is serialised behind the writer lock |
| C7 | Restore on shutdown targets a stale endpoint | Tunnel points somewhere wrong after stop | — | Restore only if `original_endpoint` is non-empty **and** the interface still exists **and** the peer still exists; otherwise skip and log | If the original endpoint is genuinely dead, restoring it makes the tunnel dead — hence `restore_on_shutdown = false` is a documented option |

## 5. Platform-specific failures

| ID | Mode | Symptom | Detection | Mitigation | Residual risk |
|---|---|---|---|---|---|
| P1 | Windows: WireGuardNT driver updated or reinstalled | Adapter handle invalid; enumeration changes | `SetEndpoint` fails | Re-enumerate the adapter on failure; retry once; then `Halted` | Driver major-version changes could alter IOCTL layout — pin the validated driver version in docs |
| P2 | Windows: adapter not present or not started | Enumeration finds nothing | `SetupDiGetClassDevs` yields no started device | `Halted` with a clear message naming the tunnel | — |
| P3 | Windows: service running as a non-privileged account | IOCTL denied | `ERROR_ACCESS_DENIED` | `Halted`; log the required account | — |
| P4 | Windows: network profile change resets routes | Tunnel appears down | `NotifyRouteChange2` | Grace period, re-evaluate | — |
| P5 | Android: Doze delays the check timer | Checks run late | Wall-clock gap between cycles | Treat delay as "no evidence"; never count a delayed cycle as a failure | Detection latency increases under Doze |
| P6 | Android: system kills the VPN or restarts it | Tunnel recreated; peer state reset | `onStateChange` / backend state | Re-initialise from the new tunnel; re-read the endpoint | A restart already happened; we can only minimise the next one |
| P7 | Android: Wi-Fi ↔ cellular transition | Handshakes fail briefly | `ConnectivityManager` callback | Grace period, immediate re-evaluate, re-resolve DNS | Android may not deliver the callback promptly |
| P8 | Android: `disableRoaming` set on peer creation | Peer does not roam; endpoint stays wherever we put it | Observed endpoint never changes unexpectedly | None needed — this is upstream's intended behaviour; document it | Divergence from desktop behaviour must be documented for users |
| P9 | Linux: kernel module vs userspace implementation | Different observation semantics | `wgctrl-go` returns the kernel client first | Adapter uses whichever client answers; no assumption about which | Behaviour differences between kernel and `wireguard-go` in edge cases |
| P10 | Linux: netlink buffer overflow under many peers | `Read` fails intermittently | `ENOBUFS` | Retry once with a larger buffer; then report `LOCAL_ERROR` | Not expected at MVP scale |

## 6. Security-relevant failures

| ID | Mode | Mitigation | Residual risk |
|---|---|---|---|
| S1 | Attacker edits companion config to add a candidate | File permission check at startup; documented operational control | Root compromise is out of scope |
| S2 | DNS poisoning redirects a hostname candidate | Handshake fails → candidate marked failed → engine moves on | If the attacker can complete handshakes they hold the key; nothing here can help |
| S3 | Attacker observes endpoints in logs | `redact_endpoints`; logs `0600` | Metadata leakage if the operator disables redaction |
| S4 | Endpoint redirection to a hostile network path | Traffic cannot be decrypted without keys | Traffic analysis and DoS remain possible |
| S5 | Status endpoint leaks key material | Status is an explicit projection, never a serialised `wgtypes.Device`; test enforces | A future contributor could add a field; the test is the control |

## 7. Accepted limitations

Stated openly so they are not discovered as bugs:

1. **A dead endpoint cannot be proven dead while its session keys are valid**, unless a
   tunnel-level target replies. This is a property of the protocol, not of this design.
   ([health §5](health-monitoring.md#5-detection-latency-bounds))
2. **A candidate cannot be tested before committing to it.** Failback is therefore an attempt
   with rollback, and can briefly interrupt the tunnel.
   ([health §2.4](health-monitoring.md#24-consequence))
3. **Roaming can undo a switch.** If the peer is reachable at a different address, WireGuard
   will use it. The engine adopts rather than fights this.
   ([endpoint-manager §4](endpoint-manager.md#4-reconciliation-with-roaming))
4. **Android cannot be a companion app.** It requires patching the `tunnel` library.
   ([platform §3](../research/platform-capabilities.md#3-android))
5. **DNS is unauthenticated.** Poisoning can cause failover to a dead endpoint, not a
   compromise.
6. **No sub-second failover.** The floor is the probe cycle, and in `keepalive` mode the
   protocol's 180 s key lifetime.
