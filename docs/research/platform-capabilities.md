# Platform capabilities: changing a peer endpoint on a live tunnel

This document answers one question per platform: **what is the supported way to change one
peer's endpoint without restarting the tunnel?** It records what exists, what it costs, and
what is missing.

Revisions cited are pinned in [`upstream-analysis.md`](upstream-analysis.md#evidence-status).

## Summary matrix

| | Linux | Windows | Android |
|---|---|---|---|
| Tunnel runs in | kernel module (`wireguard.ko`) | kernel driver (WireGuardNT) | app process (`wireguard-go`, `libwg-go.so`) |
| Control surface | generic netlink | device IOCTL | in-process Go API / UAPI socket |
| Endpoint write | `WG_CMD_SET_DEVICE` | `IOCTL_SET` | `device.IpcSet` |
| Accessible from a separate process | yes | yes (device handle) | **no** — in-process only |
| Privilege required | `CAP_NET_ADMIN` | Administrators / LocalSystem | none beyond the app |
| Restart needed | no | no | no, **if** the library is patched |
| Practical Go access | `wgctrl-go` → `wglinux` | `wgctrl-go` → `wgwindows` | requires patching `tunnel` |

The three rows are not equivalent. Linux and Windows are "write a message to a privileged
interface". Android is "call a function inside the process that owns the tunnel".

---

## 1. Linux

### Architecture

The interface is a kernel device created with `ip link add dev wg0 type wireguard`. Control
happens over the `wireguard` generic netlink family, version 1
(`WG_GENL_NAME "wireguard"`, `WG_GENL_VERSION 1`), commands `WG_CMD_GET_DEVICE` and
`WG_CMD_SET_DEVICE`.

Source: [`include/uapi/linux/wireguard.h`](https://github.com/torvalds/linux/blob/master/include/uapi/linux/wireguard.h).

### Endpoint write

Two supported routes.

**(a) `wg(8)` subprocess**

```bash
wg set wg0 peer <base64-public-key> endpoint 203.0.113.7:51820
```

Cheap to implement, but: spawns a process per switch, requires the binary to exist, returns
only a coarse exit code, and makes the peer's public key a command-line argument (visible in
`/proc/<pid>/cmdline` to the same user, and in audit logs). Acceptable as a fallback, not as
the primary mechanism.

**(b) Netlink via `wgctrl-go`** — preferred.

```go
c, _ := wgctrl.New()
defer c.Close()
err := c.ConfigureDevice("wg0", wgtypes.Config{
    Peers: []wgtypes.PeerConfig{{
        PublicKey:  peerPub,
        UpdateOnly: true,
        Endpoint:   &net.UDPAddr{IP: ip, Port: 51820},
    }},
})
```

`wgctrl-go` is pure Go (no cgo), MIT licensed, and encodes only the non-nil fields. On Linux
it speaks netlink directly through `github.com/mdlayher/netlink`.

Source: [`internal/wglinux/configure_linux.go`](https://github.com/WireGuard/wgctrl-go/blob/master/internal/wglinux/configure_linux.go).

### Reading state

`c.Device("wg0")` returns `wgtypes.Device` with, per peer: `Endpoint`,
`LastHandshakeTime`, `ReceiveBytes`, `TransmitBytes`, `AllowedIPs`, `PersistentKeepaliveInterval`.
This is everything the health monitor needs, in one netlink round trip.

### Privileges

- Creating/removing the interface, setting addresses and routes: `CAP_NET_ADMIN`.
- Reading device state: unprivileged read of the generic netlink family is possible, but
  peer enumeration is restricted on some kernels. **[unverified]** for the exact kernel
  version matrix; design assumes `CAP_NET_ADMIN` for both read and write and does not rely
  on unprivileged reads.

Recommended model: a `systemd` service with `AmbientCapabilities=CAP_NET_ADMIN`,
`CapabilityBoundingSet=CAP_NET_ADMIN`, `NoNewPrivileges=yes`, plus hardening directives. The
`wg-quick@.service` unit is the reference for lifecycle and ordering:

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/wg-quick up %i
ExecStop=/usr/bin/wg-quick down %i
ExecReload=/bin/bash -c 'exec /usr/bin/wg syncconf %i <(exec /usr/bin/wg-quick strip %i)'
```

Source: [`src/systemd/wg-quick@.service`](https://github.com/WireGuard/wireguard-tools/blob/master/src/systemd/wg-quick%40.service).

Our daemon must **not** own the interface lifecycle in the MVP. It attaches to an interface
that already exists and only ever writes `endpoint`. That keeps it composable with
`wg-quick`, NetworkManager, and hand-rolled `ip` scripts.

### DNS re-resolution

`wg-quick` sets `WG_ENDPOINT_RESOLUTION_RETRIES=infinity` in its unit and `wg(8)` retries
resolution for non-permanent errors
([`wg.8`](https://github.com/WireGuard/wireguard-tools/blob/master/src/man/wg.8)).
Our daemon resolves independently and applies the result, so it does not depend on this.

### Verdict

Fully feasible with a documented, stable, unprivileged-language API. This is the reference
platform and the one the MVP targets.

---

## 2. Windows

### Architecture (from source)

```text
wireguard.exe (manager, UI, unprivileged)
      |  anonymous pipes carrying a gob-encoded RPC
      v
WireGuard tunnel service (runs as LocalSystem, one service per tunnel)
      |  wireguard.dll  (WireGuardCreateAdapter / WireGuardOpenAdapter / ...)
      v
WireGuardNT kernel driver
```

Evidence:

- The manager's IPC server receives anonymous pipe handles, not a named pipe:
  `manager.IPCServerListen(reader, writer, events *os.File, elevatedToken windows.Token)`
  ([`manager/ipc_server.go`](https://github.com/WireGuard/wireguard-windows/blob/master/manager/ipc_server.go)),
  and `main.go` reconstructs them with `pipeFromHandleArgument(os.Args[2])`
  ([`main.go`](https://github.com/WireGuard/wireguard-windows/blob/master/main.go)).
- The service applies configuration exactly once, at startup:
  `err = adapter.SetConfiguration(config.ToDriverConfiguration())`
  ([`tunnel/service.go`](https://github.com/WireGuard/wireguard-windows/blob/master/tunnel/service.go)).
- The driver API is a flat C surface exported by `wireguard.dll`:
  `WireGuardCreateAdapter`, `WireGuardOpenAdapter`, `WireGuardCloseAdapter`,
  `WireGuardGetAdapterLUID`, `WireGuardSetConfiguration`, `WireGuardGetConfiguration`
  ([`driver/driver_windows.go`](https://github.com/WireGuard/wireguard-windows/blob/master/driver/driver_windows.go)).

### The important negative result

**The Windows tunnel service does not support changing configuration at runtime.** After
startup it enters a service control loop handling only `Stop`, `Shutdown` and `Interrogate`.
There is no UAPI socket and no "reload configuration" command. The manager IPC protocol is a
private gob RPC whose method set is start/stop/state/delete — not peer configuration.

Consequently, the two obvious approaches are both wrong:

- **Running `wg.exe`** — there is no `wg.exe` on Windows in the WireGuard for Windows
  distribution; `wg(8)` is a Unix tool.
- **Editing the `.conf` file** — the service does not watch it. Editing the file changes
  nothing until the tunnel is stopped and started, which tears down the interface, its
  addresses, its routes, and the `VpnService`-equivalent state.

### The correct mechanism

`WireGuardSetConfiguration` accepts a partial update. The peer flags include exactly what we
need:

```c
type PeerFlag uint32

const (
	PeerHasPublicKey           PeerFlag = 1 << 0
	PeerHasPresharedKey        PeerFlag = 1 << 1
	PeerHasPersistentKeepalive PeerFlag = 1 << 2
	PeerHasEndpoint            PeerFlag = 1 << 3
	PeerReplaceAllowedIPs      PeerFlag = 1 << 5
	PeerRemove                 PeerFlag = 1 << 6
	PeerUpdateOnly             PeerFlag = 1 << 7
)
```

Source: [`driver/configuration_windows.go`](https://github.com/WireGuard/wireguard-windows/blob/master/driver/configuration_windows.go)
and identically in
[`wgctrl-go/internal/wgwindows/internal/ioctl/configuration_windows.go`](https://github.com/WireGuard/wgctrl-go/blob/master/internal/wgwindows/internal/ioctl/configuration_windows.go).

So `PeerHasEndpoint | PeerUpdateOnly` with a single `PublicKey` is a peer-scoped endpoint
update — the same semantics as Linux.

### Two ways to reach it

**(a) `wgctrl-go`'s `wgwindows` client** — preferred.

`wgctrl-go` builds a `wgwindows` client on Windows
([`os_windows.go`](https://github.com/WireGuard/wgctrl-go/blob/master/os_windows.go)). It
enumerates WireGuardNT adapters through SetupAPI (device class `Net`, enumerator
`SWD\WireGuard`, falling back to `ROOT\WIREGUARD` on Windows 7), reads the adapter's device
interface path, and issues `DeviceIoControl(IOCTL_SET)` with the flag structure above.

Source: [`internal/wgwindows/client_windows.go`](https://github.com/WireGuard/wgctrl-go/blob/master/internal/wgwindows/client_windows.go).

This means **the same `ConfigureDevice` call works on Windows as on Linux**, with no
dependency on the WireGuard for Windows application, its service, or its private IPC. It
requires only the driver to be installed and the process to hold an open handle to the
adapter device.

Caveats to validate:

- `wgctrl-go` has no Windows CI job (its workflows cover Linux and static analysis). The
  Windows client is source-correct but not continuously exercised upstream. Treat it as
  supported-by-inspection until milestone 4 proves it.
- Adapter enumeration requires the device to be present and started
  (`DN_DRIVER_LOADED|DN_STARTED`).

**(b) `wireguard.dll` directly** — fallback if `wgctrl-go` proves insufficient.

Import `WireGuardOpenAdapter` / `WireGuardSetConfiguration` / `WireGuardGetConfiguration`
from the DLL installed with WireGuard for Windows, or embed it as `wireguard-windows` does
with `driver/memmod`. This is a Go `syscall` call, not a subprocess. It duplicates what
`wgctrl-go` already does, so it exists only as a contingency.

### Privileges

Opening the adapter device and issuing configuration IOCTLs requires **Administrators**
(or LocalSystem). The recommended shape is a small Windows service running as LocalSystem
that hosts the engine, with an unprivileged CLI that talks to it over a named pipe — the
same split WireGuard for Windows uses. Do not require the user to run the CLI elevated.

### Windows-specific behaviours to handle

- **Network changes.** WireGuard for Windows has `tunnel/interfacewatcher.go`,
  `tunnel/mtumonitor.go` and `tunnel/winipcfg/route_change_handler.go` because interface
  metrics, MTU and routes change on Wi-Fi/Ethernet transitions. Our health monitor must
  suspend failure counting while the local network is in flux, or it will fail over for no
  reason. On Windows this is detected with `NotifyIpInterfaceChange` /
  `NotifyRouteChange2` (wrapped by `winipcfg`).
- **Roaming.** WireGuardNT updates the peer endpoint from authenticated packets, like the
  other implementations. **[unverified]** — validate in milestone 4, because it determines
  whether the observed endpoint can be trusted as "our" write.
- **arm64.** `driver/memmod` has `memmod_windows_arm64.go`, and the official installer ships
  arm64. Our packaging may target amd64 first.

### Verdict

Feasible, and cleaner than expected: `wgctrl-go` provides the same abstraction as Linux. The
real work is packaging (a LocalSystem service), privilege separation, and validating the
driver path under load. The incorrect approaches (editing config, running `wg.exe`) must be
documented as incorrect.

---

## 3. Android

### Architecture (from source)

```text
WireGuard Android app (single APK)
  └─ VpnService (GoBackend.VpnService)            <- owns the only VPN slot
       └─ GoBackend
            └─ JNI: wgTurnOn / wgTurnOff / wgGetConfig / wgGetSocketV4/V6 / wgVersion
                 └─ libwg-go.so  (c-shared Go build of wireguard-go)
                      └─ device.Device + tun fd from VpnService
```

Evidence: `GoBackend` declares exactly these natives
([`tunnel/src/main/java/com/wireguard/android/backend/GoBackend.java`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/src/main/java/com/wireguard/android/backend/GoBackend.java)),
and the Go side exports them
([`tunnel/tools/libwg-go/api-android.go`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/tools/libwg-go/api-android.go)).

### The critical gap

There is **no** `wgSetConfig` / `wgIpcSet` native. The only way to apply configuration is:

```java
currentTunnelHandle = wgTurnOn(tunnel.getName(), tun.detachFd(), goConfig);
```

`wgTurnOn` creates a `device.Device`, calls `device.IpcSet(settings)` once, and starts it.
Changing any configuration — including one peer's endpoint — currently requires
`setStateInternal(..., State.DOWN)` followed by `wgTurnOn` again: a full tunnel restart that
destroys the tun fd and re-establishes the `VpnService`.

That is precisely the behaviour this project exists to avoid.

### The capability that makes integration possible

Despite the missing API, the tunnel **already exposes a UAPI listener**:

```go
uapiFile, err := ipc.UAPIOpen(name)
...
uapi, err = ipc.UAPIListen(name, uapiFile)
...
go device.IpcHandle(conn)
```

and the socket directory is redirected at link time:

```make
go build -tags linux \
  -ldflags="-X golang.zx2c4.com/wireguard/ipc.socketDirectory=/data/data/$(ANDROID_PACKAGE_NAME)/cache/wireguard ..."
```

Source: [`tunnel/tools/libwg-go/Makefile`](https://github.com/WireGuard/wireguard-android/blob/master/tunnel/tools/libwg-go/Makefile).

So a `set=1 ... endpoint=...` UAPI conversation is already accepted by the running device.
Two consequences:

1. The engine is technically capable of a live endpoint change **today**; only the JNI
   surface is missing. The patch is small: expose a function that calls `device.IpcSet` on
   the existing handle.
2. The socket lives in the app's private cache directory. Another application cannot reach
   it, and cannot obtain the path. Cross-app control is not an option.

### Platform constraints

- **One VPN at a time.** Android permits a single active `VpnService` per user profile. A
  second `VpnService.establish()` fails while another VPN is up
  ([`VpnService` reference](https://developer.android.com/reference/android/net/VpnService)).
  A companion app therefore cannot run its own tunnel alongside the WireGuard app. This
  eliminates the "companion application" option.
- **Always-on VPN / lockdown.** The system may restart the VPN on boot and may block traffic
  outside the tunnel. Our integration must not fight the system's always-on handling.
- **Doze and background limits.** A `VpnService` with an ongoing tunnel is foreground-ish,
  but timers are still subject to Doze. Health-check intervals must not assume precise
  wake-ups; use `Handler`/`AlarmManager` semantics that tolerate delay, and treat a delayed
  check as "no evidence" rather than "failure".
- **Network transitions.** Wi-Fi ↔ cellular changes invalidate NAT mappings and often change
  the source address. The app already reacts through `ConnectivityManager`; our monitor must
  treat a network change as a reason to *re-evaluate immediately*, not to *count a failure*.
- **Battery.** A 5 s check interval with an in-tunnel ICMP probe is too aggressive for a
  phone. Android defaults must be materially slower than the desktop defaults, and
  handshake-only checks should be preferred when a health target is not configured.
- **Roaming is disabled** by `DisableSomeRoamingForBrokenMobileSemantics()` — see
  [`upstream-analysis.md`](upstream-analysis.md#8-android-deliberately-disables-roaming).

### Integration options assessed

| Option | Viable? | Reason |
|---|---|---|
| 1. Companion application | **No** | One `VpnService` slot; cannot coexist with the WireGuard app; cannot reach the app-private UAPI socket |
| 2. Fork of the official application | Yes, but expensive | Full app fork means carrying UI, translations, build, and release divergence forever; upstream changes must be merged by hand |
| 3. Patch the `tunnel` library, integrate into the official app | **Yes** | The gap is one JNI function; the UAPI surface already exists; the change is small enough to be proposed upstream |

**Recommendation: option 3.** Add a single native function to `libwg-go` and a matching
method on `GoBackend`, then host the failover engine alongside it. Details, including the
patch shape and the upstreaming path, are in
[`../architecture/android-integration.md`](../architecture/android-integration.md).

Option 2 is the fallback if upstream rejects the patch and users need the feature now. It is
a distribution decision, not an architectural one — the engine is identical.

### Verdict

Feasible, but **not** feasible as an external companion. Android requires code inside the
application that owns the VPN. This is the one platform where "do not fork" has to be
interpreted as "patch minimally and propose upstream", and where the honest answer to the
user is: *this is an integration, not a bolt-on*.

---

## 4. Cross-platform conclusion

The endpoint write itself is uniform: *partial update of one peer, endpoint attribute only,
no restart*. What differs is **who is allowed to make the call**:

- Linux: any process with `CAP_NET_ADMIN`.
- Windows: any process with an adapter handle, i.e. Administrators/LocalSystem.
- Android: only code inside the VPN-owning app process.

That single asymmetry drives the architecture recommendation in
[`../architecture/overview.md`](../architecture/overview.md): a shared engine with three
adapters, where the Android adapter is an in-process library rather than a control-plane
client.
