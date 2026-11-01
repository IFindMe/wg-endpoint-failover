# Android integration

Android is the platform where "do not fork WireGuard" meets a hard constraint. This document
explains the constraint, the three possible answers, the recommended one, and the exact shape
of the patch that recommendation requires.

## 1. The constraint

The WireGuard Android app owns the `VpnService`, and Android allows **one** active
`VpnService` per user profile. The tunnel itself runs *in that app's process*, inside
`libwg-go.so` (a `c-shared` Go build of `wireguard-go`). The only externally visible control
surface is the JNI export list of that library:

```java
private static native String wgGetConfig(int handle);
private static native int    wgGetSocketV4(int handle);
private static native int    wgGetSocketV6(int handle);
private static native void   wgTurnOff(int handle);
private static native int    wgTurnOn(String ifName, int tunFd, String settings);
private static native String wgVersion();
```

Source: [`GoBackend.java`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/src/main/java/com/wireguard/android/backend/GoBackend.java).

There is no endpoint-update entry point. Applying configuration means `wgTurnOn` with a full
config, which tears down and rebuilds the tunnel — the exact behaviour this project exists to
avoid.

Two additional facts close off the alternatives:

- The UAPI socket the running device exposes is created inside the app's private cache
  directory, injected at link time:

  ```make
  go build -tags linux \
    -ldflags="-X golang.zx2c4.com/wireguard/ipc.socketDirectory=/data/data/$(ANDROID_PACKAGE_NAME)/cache/wireguard ..."
  ```

  Source: [`tunnel/tools/libwg-go/Makefile`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/tools/libwg-go/Makefile).

  Another application cannot read that path, so a companion app cannot talk to the running
  tunnel.

- The engine already contains a live-update path internally — `device.IpcSet(settings)` is
  called once at startup and the device keeps running. The capability exists; only the export
  is missing.

## 2. The three options

### Option 1 — Companion application

Not viable. A separate app cannot run a second `VpnService` alongside the WireGuard app, and
cannot reach the app-private UAPI socket. The only way a companion could work is by hosting
its own userspace WireGuard tunnel in its own `VpnService` — which means running a second VPN
and competing for the same slot. That is not a companion; it is a replacement.

Precedent: [wgrtc](https://github.com/gutschke/wgrtc) does exactly this — its Android app is
"a WireGuard tunnel manager" that "hosts a tunnel itself (full userspace WireGuard)". That is
the cost of this option, and it is the cost this project is avoiding.

### Option 2 — Fork the official application

Viable, expensive to maintain forever. The engine would be identical to Option 3; the
difference is distribution: a full application fork must carry UI, translations, Gradle
versions, and AndroidX churn, and must merge upstream changes by hand indefinitely.

This is the **fallback** if upstream does not accept the patch and users need the feature
today. It is a packaging decision, not an architectural one.

### Option 3 — Patch the `tunnel` library and integrate into the official application

**Recommended.** The gap is a single JNI export plus a `GoBackend` method. The change is small
enough to be a legitimate upstream contribution, and if accepted, the feature ships inside
the official app with no fork at all.

## 3. Recommended approach in detail

### 3.1 The patch

One new native export in
[`tunnel/tools/libwg-go/api-android.go`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/tools/libwg-go/api-android.go):

```go
//export wgSetConfig
func wgSetConfig(handle int32, settings string) int32 {
	tunnelHandlesLock.RLock()
	t, ok := tunnelHandles[handle]
	tunnelHandlesLock.RUnlock()
	if !ok {
		return -1
	}
	if err := t.device.IpcSet(settings); err != nil {
		return -1
	}
	return 0
}
```

One matching method on `GoBackend` that builds a **minimal** UAPI body — the `set=1`
conversation for one peer's endpoint:

```text
set=1
public_key=<peer public key, base64>
endpoint=<resolved ip:port>
\n
```

Note what is *absent* from that body: `private_key`, `preshared_key`, `allowed_ips`,
`replace_peers`. The same partial-update property as the Linux netlink call holds here, and
the same invariant is enforced.

### 3.2 Why this is safe to contribute upstream

- It adds a configuration entry point, not a protocol change.
- It does not weaken authentication or roaming semantics.
- It does not expose anything new to other applications (the library is linked into the app).
- It mirrors functionality the other two platforms already have through netlink and IOCTL.
- It is small: one export, one wrapper, one `@Keep`-annotated method.

The upstreaming proposal should be written *after* the Linux MVP proves the health model, so
the patch is accompanied by evidence rather than by intent.

### 3.3 Where the engine runs

Inside the same process, alongside `GoBackend`:

```text
WireGuard Android app
 └─ GoBackend.VpnService
     ├─ GoBackend               (existing)
     │   └─ libwg-go.so
     │        ├─ wgTurnOn / wgTurnOff / wgGetConfig      (existing)
     │        └─ wgSetConfig                             (new)
     └─ failover engine (new)
          ├─ Kotlin side: ConnectivityManager callbacks, lifecycle, status UI
          └─ Go core (same module as Linux/Windows): policy + state machine
```

The Go core is the same package used on Linux and Windows. It is compiled into the app's Go
component, so the state machine exists once across all three platforms — this is the "shared
core with native bindings" (Option C) applied where the platform makes it unavoidable.

If linking the Go core into the Android Go component proves impractical during the milestone 5
spike (size, build time, or `gomobile` friction), the documented fallback is
[`gomobile bind`](https://pkg.go.dev/golang.org/x/mobile/cmd/gomobile) producing an AAR, and
the last-resort fallback is a Kotlin reimplementation against the shared conformance test
suite. These are ordered, not equivalent: each step increases the chance of behavioural
divergence.

### 3.4 What the Kotlin side owns

The Go core owns policy. Kotlin owns the platform facts it alone can observe:

| Fact | API |
|---|---|
| Network changed | `ConnectivityManager.NetworkCallback` |
| VPN became active / stopped | `VpnService` lifecycle + `GoBackend.onStateChange` |
| Probe packets (L3) | Kotlin-side ICMP/TCP, or Go-side if linked in-process |
| Status surface | Notification / app UI driven by the Go core's status snapshot |

Network-change notifications must feed the core immediately: a Wi-Fi ↔ cellular transition
causes simultaneous failure across all candidates, and counting it as endpoint failures is
the classic source of pointless flapping
([failure-modes P7](failure-modes.md#5-platform-specific-failures)).

### 3.5 Android-specific behaviour in the engine

| Behaviour | Android treatment |
|---|---|
| Roaming | Disabled upstream for peers created with an endpoint (`DisableSomeRoamingForBrokenMobileSemantics`). The engine must not assume observed-endpoint changes. |
| Check interval | 30 s default, not 5 s |
| Backoff for failback | 300 s default |
| Keepalive probe mode | Off by default (battery); prefer `target` mode |
| Doze | Timers are advisory; a late tick is "no evidence", never a failure |
| VPN restart | Re-read the live endpoint and reconcile; never assume the tunnel survived |

### 3.6 Distribution and maintenance

Three distribution paths, in increasing order of maintenance cost:

1. **Upstream accepted** — feature ships in the official app. Best outcome.
2. **Upstream rejected, patch maintained as a patch file** — build from
   `wireguard-android` plus a small patch directory in this repository, producing our own
   signed APK. The patch is a file, not a fork, so rebasing is mechanical.
3. **Full fork** — last resort; only if (2) becomes unworkable.

Options (2) and (3) must document that the resulting APK is unofficial. The WireGuard
trademark and the Android app's package name (`com.wireguard.android`) are not ours to use;
an unofficial build must be renamed.

## 4. What must not be done

| Approach | Why not |
|---|---|
| Run a second `VpnService` | Android permits only one active VPN; it would fight the WireGuard app |
| Open the app-private UAPI socket from another app | The path is in the app sandbox; attempting it is a sandbox escape, not an integration |
| Shell out to a bundled `wg` binary | No such binary, no netlink WireGuard device, and it would require root |
| Use the root path (`WgQuickBackend`) | Requires `su`; not available on production devices |
| Fork without a stated rebase policy | Unbounded maintenance; if a fork is necessary, treat it as a product, not a side effect |

## 5. Platform feasibility verdict

| Question | Answer |
|---|---|
| Can a peer's endpoint be changed without restarting the Android tunnel? | **Yes** — the engine supports it today via `device.IpcSet`; only the JNI export is missing. |
| Can that be done without modifying the official app or library? | **No.** |
| Is the required modification small and non-protocol? | **Yes** — one export and one wrapper method. |
| Is it maintainable? | **Yes, if** the patch is upstreamed or maintained as a rebased patch file. |
| Is a companion app possible? | **No** — one `VpnService` slot, private socket. |

Android is therefore feasible but is an **integration**, not a bolt-on. Saying otherwise in
the README would be dishonest, and the milestone plan treats Android as the highest-risk
platform for that reason ([implementation-plan milestone 6](../implementation-plan.md)).
