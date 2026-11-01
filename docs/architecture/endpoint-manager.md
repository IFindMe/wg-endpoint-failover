# Endpoint manager

The endpoint manager owns *policy*: which candidate should be active, when to change it, and
what to do when reality disagrees. It does not own *evaluation* (that is
[health-monitoring.md](health-monitoring.md)) and it does not own *platform calls* (that is
the adapter).

## 1. The candidate model

```text
Interface (wg0)
  └── Peer (identified by public key)
        └── EndpointCandidate[]   ordered by (priority, config order)
              ├── id            stable identifier for logs and status
              ├── host          hostname or IP literal
              ├── port
              ├── family        auto | ipv4 | ipv6
              └── runtime state (not persisted)
                    ├── resolved[]        current addresses from DNS
                    ├── resolution       OK | UNRESOLVED | STALE
                    ├── health           UNKNOWN | HEALTHY | FAILING | FAILED
                    ├── backoff_until
                    └── last_error
```

A candidate is **not** an endpoint in WireGuard's sense. It is a *logical* endpoint that may
resolve to several addresses. The distinction matters for DNS (§4).

### Identity of a peer

The peer is identified by its **public key** only. This is deliberate:

- The public key is the same identifier WireGuard uses (`WGPEER_A_PUBLIC_KEY` is mandatory and
  is the peer selector in a partial update).
- It is not secret, so it can appear in configuration, logs and status.
- It cannot be confused with a private key, because the code never has one.

The companion configuration references the interface by name and the peer by public key.
Nothing else is required, and nothing else is permitted.

## 2. Candidate selection

Selection is a pure function of candidate state. Given the ordered candidate list and the
current time:

```text
eligible(c) =
      c.resolution != UNRESOLVED
  AND c.backoff_until <= now
  AND c.health != FAILED

select_next(current) =
    first c in ordered_candidates where eligible(c) and c != current
    else none
```

Properties:

- **Deterministic.** Same inputs produce the same choice; no randomness in selection
  (randomness exists only in backoff jitter).
- **Priority-first.** A lower `priority` number always wins, regardless of how recently it
  failed — *except* while it is in backoff. This is what "prefer the primary when it is
  healthy" means in practice.
- **No starvation.** Backoff expires, so a failed primary becomes eligible again and failback
  can attempt it.

### Why `priority` and not a score

A scoring function (weight × recency × success rate) sounds better and behaves worse: it is
hard to explain, hard to test, and produces surprising switches. Operators can reason about
"1 is preferred over 2". They cannot reason about a weighted score. Determinism beats
cleverness here.

## 3. Switching

### 3.1 The single write path

There is exactly one function that writes an endpoint:

```go
func (m *Manager) apply(ctx context.Context, c *Candidate) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    if m.generation != c.generation {        // stale decision
        return ErrStale
    }
    if time.Since(m.lastSwitch) < m.minSwitchInterval {
        return ErrCooldown
    }
    if m.switchesThisHour >= m.maxSwitchesPerHour {
        m.flapping = true
        return ErrFlapping
    }
    if err := m.ctl.SetEndpoint(ctx, m.iface, m.peer, c.addr()); err != nil {
        return err
    }
    m.lastSwitch = time.Now()
    m.switchesThisHour++
    m.active = c
    m.desired = c.addr()
    return nil
}
```

Everything that wants to change the endpoint calls this. There is no second path. This is how
the invariants "no racing switches" and "no bypassing cooldown" are enforced structurally
rather than by review.

### 3.2 Generation counter

Each evaluation cycle increments a generation. A decision carries the generation it was made
under. If a newer cycle has run, the decision is discarded. This prevents a slow DNS
resolution or a slow platform call from applying an outcome that is no longer current.

### 3.3 Idempotence

If `c.addr()` already equals the observed endpoint, `apply` returns without writing and
without incrementing `switchesThisHour`. This is common: roaming may have already moved the
peer to the candidate we were about to select, and a failback attempt may target the endpoint
that is already active.

### 3.4 Probation

After a successful write, the manager does not declare success. It enters `Probation` and
requires `recovery_threshold` consecutive `HEALTHY` evaluations before `Stable`. If probation
fails:

- revert to the previously active candidate if one exists and is still eligible;
- otherwise proceed to the next candidate;
- multiply the failed candidate's backoff.

This is the direct consequence of "you cannot test a candidate without committing to it"
([health §2.4](health-monitoring.md#24-consequence)).

## 4. Reconciliation with roaming

WireGuard will overwrite our endpoint whenever it receives an authenticated packet from a
different source address ([upstream §5](../research/upstream-analysis.md#5-roaming-silently-overwrites-a-manually-set-endpoint)).
The manager must treat this as normal, not as an error.

Every cycle compares `observed` with `desired`:

| Observed | Interpretation | Action |
|---|---|---|
| equals `desired` | Normal | none |
| is one of our candidates, ≠ `desired` | Peer roamed to a known candidate | **Adopt it.** Set `active` to that candidate, log at `info`. Do not fight it. |
| is unknown, but a handshake is current | Peer roamed to an address we do not manage (e.g. a dynamic address behind NAT) | Record it, keep policy, re-assert `desired` after `roam_reassert_after` (default 300 s) |
| is unknown and no handshake is current | Possibly an attacker-influenced address, or stale state | Re-assert `desired` immediately if we have a healthy active candidate; log at `warn` |
| is empty | No endpoint known yet | Normal at startup |

### Why adopt rather than re-assert

If the peer moved and is reachable at its new address, re-asserting our preferred endpoint
would break a working tunnel to satisfy a policy preference. Availability beats preference.
Adoption is also the honest reading of WireGuard's design: the endpoint *is* "the most recent
source address of authenticated packets".

### Why re-assert at all

Because an unknown observed endpoint could be the result of an off-path attacker injecting
packets — except that this is not possible: roaming only happens on *correctly authenticated*
packets. An unknown endpoint therefore means the peer genuinely sent from there. Re-assertion
is a policy decision ("we want traffic to go to the configured address"), not a security
measure, and the delay is deliberately long.

**Security note.** This is worth stating explicitly because it is often misunderstood:
endpoint redirection by a third party does **not** allow decryption or impersonation. It
requires valid handshake keys. The worst case is a denial of service against a tunnel whose
endpoint we keep changing. See [security.md](security.md#3-what-an-attacker-can-and-cannot-do).

## 5. Original endpoint and shutdown behaviour

Requirements: preserve the original endpoint, restore a predictable state on shutdown, never
corrupt the user's WireGuard configuration.

### At startup

1. Read the peer's current endpoint. Record it as `original_endpoint`.
2. If the endpoint is already one of our candidates, note it and start there.
3. If it is not in the candidate list, keep it as `original_endpoint` but still select a
   candidate — the candidate list is authoritative while the daemon runs.

### On clean shutdown (SIGTERM/SIGINT, service stop)

1. Enter `Restoring`.
2. Stop accepting new evaluations.
3. Write `original_endpoint` back, **if** `restore_on_shutdown = true` (default) **and**
   `original_endpoint` is non-empty.
4. Restore `persistent_keepalive_interval` if the engine changed it.
5. Flush status and exit 0.

### Why restore is optional

Some operators want the tunnel to stay on the last working endpoint after the daemon stops,
because restoring a dead original endpoint would break connectivity on daemon restart. Both
behaviours are legitimate; the default (restore) matches the requirement "restore a
predictable state", and the alternative is a documented configuration choice.

### The configuration file is never touched

The engine never reads or writes `/etc/wireguard/*.conf`, the Windows `conf/` store, or any
Android `Config` object. It only calls the platform's live configuration API. There is no
code path that could corrupt a stored configuration, because there is no code that opens one.

Runtime state (original endpoint, keepalive override, counters) lives in the daemon's memory
and, for crash recovery, in a small state file described in
[failure-modes.md](failure-modes.md#3-daemon-crash-and-restart).

## 6. Status model

Status is a read-only projection. It must be sufficient to answer "why is it on this
endpoint?" without reading logs.

```json
{
  "version": "0.1.0",
  "interface": "wg0",
  "peer": "xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=",
  "state": "STABLE",
  "active": {
    "id": "primary",
    "address": "203.0.113.7:51820",
    "priority": 1,
    "health": "HEALTHY",
    "consecutive_ok": 14
  },
  "desired_endpoint": "203.0.113.7:51820",
  "observed_endpoint": "203.0.113.7:51820",
  "roaming_detected": false,
  "original_endpoint": "198.51.100.9:51820",
  "switches_last_hour": 0,
  "flapping": false,
  "keepalive_override": null,
  "candidates": [
    {
      "id": "primary",
      "host": "endpoint-a.example.com",
      "port": 51820,
      "priority": 1,
      "resolution": "OK",
      "resolved": ["203.0.113.7"],
      "health": "HEALTHY",
      "consecutive_fail": 0,
      "backoff_until": null,
      "last_error": null
    },
    {
      "id": "backup-1",
      "host": "173.1.1.40",
      "port": 51820,
      "priority": 2,
      "resolution": "OK",
      "resolved": ["173.1.1.40"],
      "health": "UNKNOWN",
      "consecutive_fail": 0,
      "backoff_until": null,
      "last_error": null
    }
  ],
  "last_transition": {
    "at": "2026-11-01T09:14:02Z",
    "from": "SUSPECT",
    "to": "SWITCHING",
    "reason": "failure_threshold reached (3/3)"
  }
}
```

Properties:

- **No key material.** Public key only. The JSON schema is part of the public interface and is
  versioned.
- **`desired` vs `observed` are both present.** A single "endpoint" field would hide roaming.
- **`last_transition.reason` is human-readable** and is what makes support tractable.

CLI surface (milestone 2):

```text
wgefctl status   [--interface wg0] [--json]
wgefctl validate <config.toml>
wgefctl candidates [--interface wg0]
wgefctl events   [--follow]
```

## 7. Logging

Structured, one event per line, stable field names.

| Level | Event |
|---|---|
| `info` | state transitions, endpoint writes (from → to, reason), roaming adoption, DNS change |
| `warn` | unknown observed endpoint, resolution failures, stale DNS, flapping detected |
| `error` | interface missing, permission denied, all candidates failed |
| `debug` | every evaluation with inputs and result (never enabled by default) |

**Never logged:** private keys, preshared keys, packet contents, `wg showconf` output. Peer
public keys and endpoints are logged; endpoints may be considered sensitive metadata, so a
`redact_endpoints` option exists for shared environments.

Log lines are emitted *before* the action and *after* its outcome, so a crash leaves evidence
of what was attempted.

## 8. Interface summary

```go
type Manager struct { /* ... */ }

func New(cfg config.Interface, ctl adapter.Controller, dns *dns.Resolver, log *slog.Logger) (*Manager, error)
func (m *Manager) Run(ctx context.Context) error          // blocking; returns on ctx cancel
func (m *Manager) Status() status.Snapshot
func (m *Manager) Reload(cfg config.Interface) error      // SIGHUP; validated before apply
```

`Reload` validates the new configuration completely before applying any part of it. A failed
reload leaves the running configuration untouched and logs the reason. There is no partial
reload.
