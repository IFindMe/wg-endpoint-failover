# Implementation plan

Dependency-ordered milestones. Each has objectives, required components, dependencies, tasks,
acceptance criteria, tests and risks.

**Rule applied throughout:** a milestone is not complete until its acceptance criteria are
demonstrated with recorded evidence. Platform support is never claimed from design alone.

```text
M1 Repository foundation
      |
      v
M2 Linux MVP  -----------------------> M3 Automated failover tests
      |                                      |
      +--------------------------------------+-------> M4 Windows integration
      |                                                    |
      |                                                    v
      +-------------------------------------------> M5 Android integration
                                                             |
                                                             v
                                                  M6 Packaging and release automation
                                                             |
                                                             v
                                                  M7 Documentation and stable release
```

M3 depends on M2, not on M4/M5 — the test harness is Linux-only and must exist before the
portable platforms are touched, because it is what proves the policy is correct.

---

## Milestone 1 — Repository foundation

### Objectives

A repository that builds, lints and tests correctly, and that cannot accidentally leak a
secret or accept a malformed contribution.

### Required components

Go module, lint configuration, CI workflows, repository templates, branch protection.

### Dependencies

None.

### Tasks

1. Initialise the Go module (`github.com/IFindMe/wg-endpoint-failover`), Go 1.24+.
2. Add `internal/config`, `internal/engine`, `internal/health`, `internal/dns`,
   `internal/status`, `internal/adapter` skeletons — packages with types and tests, not empty
   directories.
3. Wire `golangci-lint` with a committed configuration; run it in CI.
4. Add `govulncheck` to CI.
5. Add the `invariant` test package: it asserts that no type in `internal/status`, `internal/dns`
   or the log path embeds `wgtypes.Device`, and that no exported field name matches
   `(?i)(private|preshared).*key` except an allowlist.
6. Add Dependabot configuration for Go and GitHub Actions.
7. Enable branch protection on `main`: required PR, required status checks, linear history.
8. Enable GitHub private vulnerability reporting.

### Acceptance criteria

- `go build ./...`, `go test ./...`, `golangci-lint run` and `govulncheck ./...` pass in CI on
  a clean checkout, on Linux amd64 and arm64.
- The invariant test fails when a deliberately broken fixture embeds `wgtypes.Device`.
- Branch protection rejects a direct push to `main`.

### Tests

Unit tests for the invariant package; CI green on a trivial package.

### Risks

| Risk | Mitigation |
|---|---|
| The invariant test is written after the status code exists and codifies a leak | Write the test first, with the fixture; it must fail before the implementation is correct |
| Over-scaffolding produces empty modules | Only packages with types and tests are added; no placeholder for Android yet |

---

## Milestone 2 — Linux MVP

### Objectives

A working Linux daemon that reads companion configuration, monitors one peer, and performs
correct, safe endpoint switching with failback.

### Required components

- `internal/config` — TOML parsing, strict validation, defaults
- `internal/adapter/controller.go` — the interface
- `internal/adapter/linux` — `wgctrl-go`
- `internal/dns` — resolver with cache, multi-address, `UNRESOLVED`
- `internal/health` — stimulus, evaluation, `Result` classification
- `internal/engine` — state machine, switching policy, reconciliation, restore
- `internal/status` — snapshot and rendering
- `cmd/wg-endpoint-failover`, `cmd/wgefctl`
- systemd unit, man pages

### Dependencies

M1.

### Tasks

1. Configuration: parser, validator with every rule in
   [configuration.md §5](architecture/configuration.md#5-validation-rules), defaults per
   platform, and `wgefctl validate`.
2. Linux adapter: `Read` and `SetEndpoint` via `wgctrl-go`, with `UpdateOnly: true` and
   `Endpoint` as the only non-nil field.
3. DNS: cache, refresh interval, multi-address sub-candidates, `UNRESOLVED`/`STALE` states,
   IPv4/IPv6 policy.
4. Health: `target` mode (ICMP then TCP), `keepalive` mode with recorded-and-restored
   override, `passive` mode; `HEALTHY`/`FAILURE`/`INCONCLUSIVE`/`LOCAL_*` classification.
5. Engine: full state machine with all transitions and guards from
   [health-monitoring.md §8](architecture/health-monitoring.md#8-state-machine); single
   `apply()` writer; generation counter; per-candidate backoff with jitter; global switch rate
   limit; reconciliation with roaming; restore on shutdown.
6. Local network change detection on Linux (rtnetlink).
7. Instance lock (`flock`).
8. Status: JSON and human-readable; `last_transition.reason`.
9. CLI: `status`, `validate`, `candidates`, `events`.
10. systemd unit with the hardening from
    [security.md §4.2](architecture/security.md#42-linux-systemd-hardening); validate a
    non-root user with `AmbientCapabilities` and prefer it if it works.
11. Config file permission check (root-owned, not group/other-writable).
12. Man pages: `wg-endpoint-failover.8`, `wg-endpoint-failover.conf.5`, `wgefctl.1`.

### Acceptance criteria

- With a real interface and two candidates, killing the primary causes a switch within the
  documented bound, with **peer keys, preshared key, `AllowedIPs` and interface addresses
  byte-identical** before and after (verified by `wg show dump` diff with keys redacted).
- The interface never goes down during a switch.
- With the primary healthy, `state == STABLE` and no writes occur over 10 minutes.
- An idle tunnel produces **no** failover over 30 minutes (the C4 regression test).
- Stopping the daemon restores the original endpoint; `restore_on_shutdown = false` does not.
- The daemon refuses to start on a world-writable config.
- `wgefctl validate` reports every rule violation in
  [configuration.md §5](architecture/configuration.md#5-validation-rules) with file, field and
  an actionable message.

### Tests

- Unit: config validation table-driven over every rule; DNS with a fake resolver; engine state
  machine with a fake controller and a fake clock (no real time in unit tests).
- Integration: network namespaces, virtual WireGuard interfaces, described in milestone 3.

### Risks

| Risk | Mitigation |
|---|---|
| Detection latency is worse than the documented bound | Record the actual latency in the acceptance run; correct the documentation rather than the measurement |
| `keepalive` mode interferes with a user's own keepalive settings | Record, restore, log; make it disableable |
| `wgctrl-go` returns `os.ErrNotExist` when a userspace implementation is in play | Test with both kernel and `wireguard-go` interfaces |

---

## Milestone 3 — Automated failover tests

### Objectives

A hermetic Linux integration harness that proves the policy against real WireGuard, and runs
in CI without root on the host.

### Required components

`test/integration/` with a network-namespace harness; a Go test driver; CI job using
`sudo` inside the GitHub-hosted runner.

### Dependencies

M2.

### Tasks

1. Harness: create `client` and `server` namespaces joined by a veth pair, load `wireguard`,
   create `wg0` on each side with known keys, and expose controls to kill/restore
   connectivity per endpoint.
2. Multi-endpoint server: three IPs on the server side so one interface has three reachable
   candidates.
3. Test matrix — each must assert key/state invariants:

   | Scenario | Assertion |
   |---|---|
   | Primary available | Stays on primary; no writes |
   | Primary unavailable | Switches to the best backup within bound |
   | Backup available | Established traffic continues across the switch |
   | All unavailable | `BACKOFF`; tunnel not torn down; recovery works |
   | Primary recovers | Fails back after cooldown; no flap during recovery |
   | Repeated interruptions (10×) | Switch count ≤ expected; never exceeds `max_switches_per_hour` |
   | DNS changes | Switch to the new address; no failover for unrelated peers |
   | Multiple DNS addresses | Tries the next address before declaring the endpoint down |
   | IPv4/IPv6 combinations | Falls back across families; never races both |
   | Peer roams (server changes source address) | Engine adopts; no fight |
   | Daemon restart (clean) | Original endpoint restored |
   | Daemon restart (killed -9) | Reconciles from live state; no interruption |
   | Simultaneous failover on two peers | Each resolves independently; no cross-talk |
   | Invalid configuration | Refuses to start; nothing written |
   | Unreachable health target | `INCONCLUSIVE`, no failover |
   | Probe target not in `AllowedIPs` | Startup warning |

4. Invariant assertion in every switch test: capture `wg show <if> dump`, redact keys, diff
   before/after, and fail on any non-endpoint difference.
5. Soak test (nightly, not on PR): 60 minutes with randomised connectivity toggles; assert no
   `FLAPPING` and no leaked state.

### Acceptance criteria

- All scenarios pass in CI on `ubuntu-24.04` as an unprivileged user invoking `sudo` only for
  namespace setup.
- The suite is deterministic: no scenario depends on wall-clock sleep longer than the
  documented bound.
- A deliberately broken build (a build that writes `AllowedIPs` on switch) makes the suite
  fail.

### Risks

| Risk | Mitigation |
|---|---|
| Flaky timing on shared runners | Use generous bounds and a monotonic clock; retry-once policy for the suite, never per assertion |
| `wireguard` module unavailable in the runner | Fall back to `wireguard-go` in userspace; both are exercised |
| The suite passes while the product is broken | Include a mutation check: at least one negative test must fail against a known-bad variant |

---

## Milestone 4 — Windows integration

### Objectives

The same engine on Windows, driving WireGuardNT through device IOCTLs, hosted in a
LocalSystem service.

### Required components

`internal/adapter/windows`, `internal/netchange_windows.go`, a Windows service host, an
unprivileged CLI talking over a named pipe, an MSI.

### Dependencies

M2 and M3 (the policy must already be proven before the second platform is attempted).

### Tasks

1. Spike, before any implementation: on a Windows machine with WireGuard for Windows
   installed, call `wgctrl-go`'s `ConfigureDevice` with `UpdateOnly` + `Endpoint` on a running
   tunnel, and measure whether an in-flight session survives. Record the result in
   `docs/research/platform-capabilities.md` — this closes the `[unverified]` marker on
   WireGuardNT roaming and on runtime endpoint updates.
2. Windows adapter: `wgctrl-go`; re-enumeration and one retry when the adapter handle becomes
   invalid (driver reinstall).
3. Service host as `LocalSystem`; named pipe with an `Administrators` + `LocalSystem` security
   descriptor; **read-only** CLI.
4. Config path `%ProgramData%\wg-endpoint-failover\config.toml` with an ACL check mirroring
   the Linux permission check.
5. Local network change detection: `NotifyIpInterfaceChange`, `NotifyRouteChange2`.
6. State file and crash reconciliation.
7. MSI with WiX, matching the structure WireGuard for Windows uses
   (`installer/wireguard.wxs`), including service registration and uninstall that restores
   the original endpoint.

### Acceptance criteria

- The same test scenarios from milestone 3 (primary down, all down, failback, restart,
  roaming) pass on Windows, with the same invariant assertion.
- The CLI works without elevation; the service performs all writes.
- Uninstalling leaves the tunnel on the original endpoint and removes all files.
- amd64 is validated; arm64 is built and smoke-tested or explicitly documented as untested.

### Risks

| Risk | Mitigation |
|---|---|
| `wgctrl-go`'s Windows client is not CI-exercised upstream | Keep the adapter thin; our own tests are the gate; pin the validated WireGuard for Windows version in the docs |
| Driver IOCTL layout changes between versions | Detect the running driver version at startup; refuse a major-version mismatch rather than sending a possibly-wrong structure |
| The driver rejects an endpoint update on an Up adapter | The spike in task 1 answers this before any code is written; if it fails, this milestone stops and the design is revisited |

---

## Milestone 5 — Android integration

### Objectives

Live endpoint updates in the official-app codebase, with the shared engine running in-process,
and a defensible distribution path.

### Required components

The `wgSetConfig` patch to `tunnel/tools/libwg-go/api-android.go`, a `GoBackend` wrapper, the
Go core linked into the app's Go component, a Kotlin side for network callbacks and status.

### Dependencies

M2, M3 and the Android spike below. Highest-risk milestone; deliberately last of the three
platforms.

### Tasks

1. **Spike, before committing to the milestone:** confirm that `device.IpcSet` on a running
   handle changes a peer's endpoint in a live Android tunnel without dropping the tun fd, and
   that traffic resumes. Record the result in
   [android-integration.md](architecture/android-integration.md).
2. Decide the binding approach by measurement, in this order:
   a. Go core linked into the app's Go component (preferred);
   b. `gomobile bind` AAR;
   c. Kotlin reimplementation against the shared conformance suite (last resort).
   Record which was chosen and why.
3. Implement `wgSetConfig` and the `GoBackend` wrapper.
4. Kotlin: `ConnectivityManager.NetworkCallback` feeding the core; lifecycle wiring; status
   surface.
5. Android defaults ([ADR-010](architecture/decisions.md#adr-010-platform-specific-default-parameters)).
6. Doze handling: a late tick is "no evidence", never a failure.
7. Conformance run: the shared behaviour suite executed against the Android core.
8. Write the upstreaming proposal with evidence.

### Acceptance criteria

- A live tunnel switches endpoints without the tun fd being recreated, demonstrated by
  a continuous ping across the switch.
- Key, preshared key, `AllowedIPs` and tunnel address are unchanged across the switch
  (asserted from `wgGetConfig` output with keys redacted).
- Wi-Fi ↔ cellular transition does not produce a spurious failover.
- Doze does not produce a false failure.
- The distribution path (upstream, patch, or fork) is stated in the README with its
  consequences.

### Risks

| Risk | Mitigation |
|---|---|
| The spike shows `IpcSet` does not work live on Android | The milestone stops; the fallback is `fork`, and the README must say so rather than claiming support |
| Upstream rejects the patch | Maintain a rebased patch file; build our own APK; rename the package and do not use the WireGuard trademark |
| Go core linking bloats the APK or breaks the build | Fall back to `gomobile bind`, then to Kotlin |
| Battery regression | Android defaults are materially slower; measure with the Android battery profiler and publish the result |

---

## Milestone 6 — Packaging and release automation

### Objectives

Reproducible, signed, checksummed artifacts for each shipped platform, produced only from
tags.

### Dependencies

M2 for Linux; M4 for Windows; M5 for Android.

### Tasks

1. Debian package: `dh`-based, systemd unit, man pages, `postinst` that creates the config
   directory `0750`, `prerm` that stops the service.
2. Windows MSI and service registration.
3. Android APK/AAB with release signing from GitHub Secrets.
4. Tag-triggered release workflow; SHA256 checksums; release notes generated from the
   changelog; a provenance note naming the commit.
5. Windows Authenticode signing, documented with the key-handling strategy even if a
   certificate is not yet available — the workflow must skip signing with a visible warning
   rather than silently producing an unsigned binary.
6. SBOM per artifact.

### Acceptance criteria

- A tagged commit produces every artifact, its checksum, and a release with notes.
- Installing the Debian package and the MSI on a clean machine yields a working daemon with no
  manual steps beyond writing the config.
- No workflow produces a release from a non-tag build.
- A checksum mismatch is detectable: the release lists SHA256 for every file.

### Risks

| Risk | Mitigation |
|---|---|
| Signing secrets leak | Store only in GitHub Secrets/OIDC; never in the repository; sign in an ephemeral runner |
| Artifacts are built from unreproducible inputs | Pin toolchain versions; record the Go version and the `wgctrl-go` version in the artifact metadata |

---

## Milestone 7 — Documentation and stable release

### Objectives

Documentation that matches the shipped behaviour, and a 1.0.0 that is honest about platform
support.

### Dependencies

M4, M5, M6.

### Tasks

1. User guide: installing, writing the companion config, reading status, choosing probe mode.
2. Operations guide: the systemd/Windows service hardening, log redaction, the
   `restore_on_shutdown` trade-off.
3. Update every `[unverified]` marker in the research documents with measured results, or
   delete the claim.
4. Reconcile the documentation against the implementation; every default, every validation
   rule and every state name must match.
5. Release notes and a 1.0.0 tag.

### Acceptance criteria

- No `[unverified]` markers remain, or each is explicitly documented as an accepted unknown.
- Every documented default exists in the code, checked by a test that parses the
  documentation's default table.
- The README's platform table lists only platforms that are implemented, tested and released.

### Risks

| Risk | Mitigation |
|---|---|
| Documentation drifts from behaviour | A test asserts the documented defaults match the code; the drift check is part of CI |
| Pressure to claim a platform that is only partially working | The platform table lists a platform only when its milestone acceptance criteria are met |

---

## Cross-cutting work

| Item | Where |
|---|---|
| Invariant tests (no key writes, no `wgtypes.Device` in status) | M1, re-run in every milestone |
| Conformance behaviour suite shared across platforms | M3 defines it; M4 and M5 execute it |
| Evidence recording (every acceptance run saved as a CI artifact) | Every milestone |
| ADR for each decision that changes architecture | Ongoing; see [decisions.md](architecture/decisions.md) |

## What is deliberately not planned

| Not planned | Why |
|---|---|
| Switching between peers with different public keys | Out of scope by requirement; that is a different product (see [existing-solutions §4](research/existing-solutions.md#4-privycs--adjacent-and-deliberately-out-of-scope)) |
| NAT traversal, STUN, signalling brokers | Out of scope by requirement ([ADR-011](architecture/decisions.md#adr-011-no-active-endpoint-discovery)) |
| macOS / FreeBSD / OpenBSD | `wgctrl-go` supports FreeBSD and OpenBSD, so the adapter would be nearly free — but untested platforms must not be claimed. Revisit after 1.0.0. |
| A kill switch, routing management, or DNS redirection | Firewall policy; each platform already has mechanisms |
| An automatic migration from `reresolve-dns` | Different data model; a migration guide is enough |
