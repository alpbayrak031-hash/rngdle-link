# Porting MacDPI to Windows

A build guide for a Windows version of MacDPI. "WinDPI" below is just a working
name; call it whatever you like.

It is written against MacDPI **1.3.1** (`macdpi/`, `MenuBar/`, `mcroute-agent/`)
and assumes you know that codebase. Every section says **what to keep, what to
replace, and how you know it works**. Keep the "how you know it works" checks.
Most of MacDPI's bugs were things that looked fine from the outside.

---

## 0. The big picture

### What carries over and what doesn't

| Layer | macOS today | Windows | Effort |
|---|---|---|---|
| Evasion strategies (`desync.py`, `tls.py`) | Python, stdlib | **unchanged** | none |
| Verified auto-selection, learning, re-verify (`proxy.py`) | Python | **unchanged** except paths | trivial |
| DoH resolver (`dns.py`, `tlsclient.py`) | Python | unchanged; add the CA bundle | small |
| Socket layer (`netio.py`) | `add_writer` + raw `send(flags)` | **must change** for Windows' asyncio loop | small but critical |
| Control channel (`control.py`) | Unix socket `~/.macdpi/control.sock` | loopback TCP + secret token (or a named pipe) | small |
| Network-change detection (`netcheck.py`) | `/sbin/route` | `GetBestRoute` via ctypes | small |
| Bundled runtime | python-build-standalone, universal | official **Windows embeddable Python** (x64, arm64) | small |
| Menu bar app (Swift/AppKit) | `MenuBar/*.swift` | **tray app**, C# WinForms on .NET Framework 4.8 | the big one |
| System proxy | `networksetup` + password | WinINET settings in **HKCU**, **no admin** | small |
| Start at login / keep alive | launchd LaunchAgents | HKCU `Run` key + tray supervises the engine | medium |
| "Cover All Apps" | `pf` `route-to` + `rdr`, LaunchDaemon | **WinDivert** redirector in a Windows **service** | the hard one |
| Minecraft route agent | Java agent | **unchanged** jar; only the path changes | none |
| Packaging | `.app` + DMG, ad-hoc signed | Inno Setup `Setup.exe` + zip, unsigned | medium |
| CLI | `dpictl` (bash) | `dpictl.ps1` | small |

About 80% of the engine runs on Windows as it is. Most of the real work is the
tray app, the installer and Cover All Apps.

### Recommended architecture

```
                 ┌───────────────────────────── per user, no admin ────────────────────────────┐
  Browsers ──WinINET system proxy──▶ 127.0.0.1:8881  ┌──────────────────────────────┐
  (Chrome/Edge/Firefox)                              │ engine: pythonw.exe -m macdpi │──▶ Internet
                                                     │  (same Python code as macOS)  │   (upstream sockets bind
  WinDPI.exe (tray) ──control: 127.0.0.1:<rand>+token│                               │    source ports 40000-44999)
     │  starts / restarts the engine                 └──────────────▲───────────────┘
     │  sets / restores the proxy (HKCU)                            │ reflected :443/:80/:25565
     │                                                              │
 ────┼───────────────────────────── admin, installed once ──────────┼─────────────────────────────
     │ UAC                                          ┌───────────────┴──────────────┐
     └──────────────────────────────────────────────│ "WinDPI Redirect" service     │
                                                    │ WinDPI.exe --redirect-service │
                                                    │ WinDivert: every app's TCP    │
                                                    │ 80/443 → engine listeners     │
                                                    └──────────────────────────────┘
```

The design rules are the same as on macOS, and they matter just as much:

1. **The engine never runs elevated.** Only the redirector needs admin, and it
   contains no evasion logic. This is the Windows version of "nothing but fixed
   `pfctl` commands run as root".
2. **The engine outlives the tray app.** Quitting the tray must never take
   browsing down.
3. **Fail safe.** If the system proxy is on but the engine is dead, nothing
   loads. If the redirect is on but the engine is dead, it is worse: all web
   traffic disappears. Each side watches the other (§5, §7).

### Repository layout: share the engine, don't fork it

Strategy ordering, verified selection and DoH fallback are the parts that took
live probing to get right (see the DPI findings). If you keep two copies, they
will drift apart. Make `macdpi/` cross-platform behind a small platform layer:

```
macdpi/
  plat/__init__.py      # picks posix or windows at import time
  plat/posix.py         # state dir, control transport, default route (today's code)
  plat/windows.py       # %LOCALAPPDATA%, TCP+token control, GetBestRoute, ...
  ...everything else unchanged...
WinTray/                # C# tray app + redirect service (one exe)
  WinDPI.csproj
  Program.cs  TrayApp.cs  Control.cs  SystemProxy.cs  Supervisor.cs
  Installer.cs  Paths.cs  Version.cs  Redirect/RedirectService.cs  Redirect/WinDivert.cs
windows/
  build.ps1  WinDPI.iss  dpictl.ps1
```

If you would rather keep a separate repo, copy `macdpi/` once and apply the
same edits. Re-sync whenever the macOS ladder or the DoH list changes.

---

## 1. Make the engine run on Windows

Set up a dev box with python.org's CPython 3.12 for Windows. Do the edits
below, then run the checks at the end of this section before you start on
anything else.

### 1.1 asyncio: the one change that really matters (`netio.py`)

Since Python 3.8 the default event loop on Windows is the **ProactorEventLoop**
(IOCP). It **does not support `loop.add_writer` / `remove_writer`**, and
`netio.Upstream` uses both: in `_writable()` and in the cleanup after a failed
connect.

Don't switch to `WindowsSelectorEventLoopPolicy`. The selector loop on Windows
is built on `select()`, which is limited to **512 sockets**. Cover All Apps
would hit that limit within minutes. That's the Windows version of the launchd
`RLIMIT_NOFILE=256` bug.

Stay on Proactor and change how `Upstream` sends:

```python
import sys
_WINDOWS = sys.platform == "win32"

class Upstream:
    async def send(self, data: bytes, flags: int = 0):
        if _WINDOWS:
            if flags:                       # only MSG_OOB, always 1 byte
                await self._send_sync(data, flags)
            else:
                await self.loop.sock_sendall(self.sock, data)
            return
        ...existing POSIX loop with self._writable()...

    async def _send_sync(self, data, flags):
        view = memoryview(data)
        while view:
            try:
                sent = self.sock.send(view, flags)
            except (BlockingIOError, InterruptedError):
                await asyncio.sleep(0.005)
                continue
            view = view[sent:]
```

In `Upstream.connect()`, guard the cleanup:

```python
except BaseException as exc:
    if not _WINDOWS:
        loop.remove_writer(fd)
    sock.close()
```

Why this is safe:

- `sock_sendall` on Proactor finishes only once the kernel has taken the bytes,
  so a following urgent byte still goes out in order (this matters for the
  `oob` family).
- Calling a plain synchronous `send()` on a socket that is attached to IOCP is
  allowed.
- Proactor's `sock_connect` uses `ConnectEx`, which needs a socket that is
  already bound. Our sockets are already bound to the reserved band, and for
  unbound ones CPython binds them itself.

**Segment boundaries.** Every split strategy depends on each `send()` leaving
as its own TCP segment. `TCP_NODELAY` is already set and Windows normally
honours it. You still need to **confirm it with Wireshark** (see 1.9). If two
chunks are merged into one segment, `split`, `multisplit`, `tlsfrag`,
`tinyrec` and `multirec` quietly stop working. The fix is a small default delay
on Windows (`ctx.delay = 0.002`).

`IP_TTL` / `IPV6_UNICAST_HOPS` (used by `disorder`) and `MSG_OOB` (used by
`oob`) both exist in Winsock and Python exposes them. No change is needed.

### 1.2 Reserved source-port band (`netio.py`)

- Windows' ephemeral range is **49152-65535** (check it with
  `netsh int ipv4 show dynamicport tcp`). 40000-44999 is outside it, same as on
  macOS. Keep the band.
- **Hyper-V, WSL2, Docker and Windows Sandbox reserve blocks of ports**, and
  those blocks often fall around 40000-50000. You can see them with
  `netsh int ipv4 show excludedportrange protocol=tcp`. Binding to a reserved
  port fails with `WSAEACCES` (10013). `_bind_reserved_source_port` already moves
  on after any `OSError`, but tell the `connect()` retry about it too:

  ```python
  _PORT_CLASH = (errno.EADDRINUSE, errno.EADDRNOTAVAIL, errno.EACCES)
  # and on Windows also compare exc.winerror in (10048, 10049, 10013)
  ```

  At startup, log the excluded ranges that overlap the band. If an overlap
  covers most of it, move the band (for example to 30000-34999). If you do,
  keep the redirector's filter in sync (§7). Both sides should read the band
  from one constant.
- `SO_REUSEPORT` doesn't exist on Windows. It is already wrapped in
  `try/except`, so nothing to do.

### 1.3 Listening sockets: `SO_REUSEADDR` means something else on Windows

On macOS, `reuse_address=True` only lets you rebind a port that is in
`TIME_WAIT`. On Windows, `SO_REUSEADDR` lets **a second process bind the same
port while the first is still listening**. Two engines can then share 8881 and
connections get split between them at random. Fix it in `__main__.py`:

```python
def _listen_socket(host, port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    if _WINDOWS:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_EXCLUSIVEADDRUSE, 1)
    else:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((host, port))
    return s

server = await asyncio.start_server(prox.handle, sock=_listen_socket(ctx.host, ctx.port),
                                    backlog=512)
```

Do the same for the transparent and Minecraft listeners. Because the port is
held exclusively, a second engine fails at the bind, and that becomes your
"already running" check.

Bind the proxy to **127.0.0.1 only**. A loopback bind doesn't trigger the
Windows Defender Firewall "allow access?" dialog. A `0.0.0.0` bind does, and it
also opens your proxy to the whole LAN.

### 1.4 Things that don't exist on Windows (`__main__.py`)

- `import resource` → move it inside `_raise_fd_limit` and skip that function
  on Windows. There's no per-process socket limit to raise.
- `loop.add_signal_handler` raises `NotImplementedError`. Wrap it:

  ```python
  for sig in (signal.SIGINT, signal.SIGTERM):
      try:
          loop.add_signal_handler(sig, stop.set)
      except (NotImplementedError, AttributeError):
          signal.signal(sig, lambda *_: loop.call_soon_threadsafe(stop.set))
  ```

  Windows has no SIGTERM for other processes. A `TerminateProcess` kill skips
  all clean-up. The **normal stop path is the control `shutdown` command**
  (already implemented), and the tray must use it. `learned.json` is saved
  every 20 s already, so a hard kill loses at most 20 s of learning.

### 1.5 Logging when there's no console (`config.py`)

The engine runs under **`pythonw.exe`** so no console window appears. Under
`pythonw`, `sys.stderr` and `sys.stdout` are `None`, and
`print(..., file=sys.stderr)` silently does nothing. On macOS, launchd's
`StandardErrorPath` captured the log. On Windows nothing does, so the engine
has to write its own log file:

- Add `--log-file PATH`. Open it UTF-8, line-buffered, and roll it at around
  5 MB (`macdpi.log` → `macdpi.log.1`).
- Have `Log._emit` write there. Fall back to stderr only if stderr exists.
- Install `sys.excepthook` and `loop.set_exception_handler` so a crash ends up
  in the log and doesn't vanish.

### 1.6 Paths and encoding (`config.py`, `proxy.py`, `control.py`)

- `STATE_DIR`: `%LOCALAPPDATA%\WinDPI` on Windows (`os.environ["LOCALAPPDATA"]`),
  `~/.macdpi` elsewhere. Put this in `plat/`.
- Windows' default text encoding is still the ANSI code page (cp1252/cp1254).
  Either pass `encoding="utf-8"` to every `open()` (`learned.json`,
  `bypass.txt`, the log), or always start the engine with **`-X utf8`**. Do
  both.
- `os.chmod(SOCKET_PATH, 0o600)` is meaningless on Windows. It goes away with
  the control-channel change below.

### 1.7 Control channel (`control.py`)

`asyncio.start_unix_server` isn't available on Windows. The Unix socket was
chosen so that a web page cannot reach the control surface. Keep that
guarantee with **loopback TCP plus a secret token**:

1. Bind `127.0.0.1:0` (let the OS pick a port).
2. Generate `token = secrets.token_hex(32)` at startup.
3. Write `{"port": N, "token": "...", "pid": ..., "version": ...}` to
   `%LOCALAPPDATA%\WinDPI\control.json`, atomically (write `.tmp`, then
   `os.replace`). `%LOCALAPPDATA%` is per-user, so other accounts cannot read
   it.
4. Every request must carry `"token"`. Check it with `hmac.compare_digest`.
   On a mismatch, drop the connection without replying.
5. In `proxy.py`, **refuse CONNECT/SOCKS/HTTP targets of 127.0.0.0/8, ::1 and
   `localhost`**. That's good practice anyway, and it means the proxy can't
   be used to reach its own control port.

Keep the protocol the same: one JSON object per line. The commands (`status`,
`recent`, `learned`, `set_mode`, `flush_dns`, `forget`, `selftest`, `shutdown`,
`get_bypass`/`set_bypass`) carry over. Replace `pf_ruleset` with
`redirect_config` (§7).

*Alternative:* a named pipe `\\.\pipe\WinDPI-<user SID>`. Its default ACL is
already owner-only, but Python's asyncio only reaches pipes through an
undocumented Proactor API. TCP + token is the simpler choice.

### 1.8 Default route and network changes (`netcheck.py`)

`_default_route()` runs `/sbin/route -n get default`. On Windows, use the IP
Helper API through `ctypes`. There's no subprocess, so no console window
flashes:

```python
import ctypes, socket, struct
from ctypes import wintypes

class MIB_IPFORWARDROW(ctypes.Structure):
    _fields_ = [(n, wintypes.DWORD) for n in (
        "dwForwardDest", "dwForwardMask", "dwForwardPolicy", "dwForwardNextHop",
        "dwForwardIfIndex", "dwForwardType", "dwForwardProto", "dwForwardAge",
        "dwForwardNextHopAS", "dwForwardMetric1", "dwForwardMetric2",
        "dwForwardMetric3", "dwForwardMetric4", "dwForwardMetric5")]

async def _default_route():
    row = MIB_IPFORWARDROW()
    dest = struct.unpack("<I", socket.inet_aton("1.1.1.1"))[0]
    if ctypes.windll.iphlpapi.GetBestRoute(dest, 0, ctypes.byref(row)) != 0:
        return None
    gateway = socket.inet_ntoa(struct.pack("<I", row.dwForwardNextHop))
    return gateway, row.dwForwardIfIndex
```

`_local_address()` (the UDP connect trick) and the polling `monitor()` work as
they are. Polling every 30 s also covers sleep/resume.

**General rule:** whenever any engine code spawns a process on Windows, pass
`creationflags=subprocess.CREATE_NO_WINDOW`. Without it, a console window
flashes up every time a process is started from `pythonw`.

### 1.9 CA certificates (`tlsclient.py`)

`ssl.create_default_context()` on Windows loads the system ROOT and CA stores.
The catch is that **Windows fills its root store lazily**. A root that
CryptoAPI has never needed may not be there yet on a fresh install. If that
happens, the engine's verifying probe fails, and verified selection decides a
real server is "intercepted". That's the wrong answer, and it gets cached.

Fix: bundle `certifi`'s `cacert.pem` (copy the file, nothing to install) and
always load **both** stores:

```python
ctx = ssl.create_default_context()           # Windows stores
ctx.load_verify_locations(BUNDLED_CACERT)    # + Mozilla's roots
```

Replace the macOS paths in `_CA_FALLBACKS` with the bundled path.

### 1.10 Antivirus "HTTPS scanning" breaks evasion silently

On Windows this is much more common than on a Mac. Kaspersky, ESET, Avast/AVG,
Bitdefender and others intercept TLS **inside the machine**, below the engine.
They end your connection and open their own. **Your carefully split
ClientHello is thrown away and re-sent in one clean piece** by the antivirus.
Every strategy then "fails" for reasons that have nothing to do with the
network.

- Add a startup check. If the certificate chain of a known site (for example
  `www.google.com`) has an issuer that isn't the real CA, log a warning and
  show it in the tray: "Your antivirus inspects HTTPS. Exclude WinDPI or turn
  off HTTPS scanning."
- Note it in the README.

### 1.11 Milestone checks: don't continue until all of these pass

Run these in PowerShell, from the repo:

```powershell
py -3.12 -X utf8 -m macdpi --test discord.com www.roblox.com
py -3.12 -X utf8 -m macdpi --port 8899 -v --log-file "$env:TEMP\windpi-test.log"
```

```powershell
curl.exe --proxy http://127.0.0.1:8899 -sS -o NUL -w "%{http_code}`n" https://discord.com/
```

- **Use `curl.exe`, not `curl`.** In Windows PowerShell 5.1, `curl` is an alias
  for `Invoke-WebRequest`. This is the Windows version of the zsh word-split trap.
- Send a browser User-Agent for www.minecraft.net, the same as on macOS.
- **Wireshark** (with Npcap), filter `tcp.port == 443 && tls.handshake.type == 1`:
  - with `-m split`, the ClientHello must arrive in **two TCP segments** cut
    inside the hostname;
  - with `-m tlsfrag`, there must be two TLS records;
  - with `-m recver`, the record header must read `16 03 ff`;
  - with `-m oob`, you must see the URG flag set.
- Open 300+ connections at once (any load tool, or a quick asyncio script)
  and confirm that nothing stalls. This shows you're on Proactor and not
  hitting the select() limit.
- Kill the engine with Task Manager and restart it. It must start listening
  immediately, before any network check (the v1.2.1 lesson), and `learned.json`
  must still be valid.

---

## 2. Bundle the runtime ("nothing to install")

Use the **official Windows embeddable package** from python.org:
`python-3.12.x-embed-amd64.zip` and `python-3.12.x-embed-arm64.zip`. It's about
10 MB, contains `python.exe` and **`pythonw.exe`**, and its stdlib is a
`python312.zip`. You don't need pip; the engine is stdlib-only.

Layout inside the install directory:

```
WinDPI\
  WinDPI.exe                 tray app + redirect service
  python\                    embeddable package, untouched except ._pth
    python.exe  pythonw.exe  python312.dll  python312.zip  python312._pth
  engine\macdpi\...          the engine package
  engine\cacert.pem          certifi bundle
  WinDivert.dll  WinDivert64.sys   (only for Cover All Apps, §7)
  macdpi-mcroute.jar         Minecraft route agent (§8)
```

Edit `python312._pth` so the engine is importable without `PYTHONPATH`:

```
python312.zip
.
..\engine
```

Engine command line (the Windows version of `Installer.engineArguments`):

```
"<install>\python\pythonw.exe" -X utf8 -X pycache_prefix=%LOCALAPPDATA%\WinDPI\pycache
    -m macdpi --port 8881 --wait-for-network 300 --log-file %LOCALAPPDATA%\WinDPI\macdpi.log
```

`pycache_prefix` still matters. Program Files isn't writable by the user, and
on macOS a stray `__pycache__` broke the code signature.

**Windows on ARM:** ship the arm64 embeddable package for the engine. The tray
app (net48, AnyCPU) runs natively. WinDivert only ships x86/x64 kernel
drivers, and kernel drivers aren't emulated, so **expect Cover All Apps to be
unavailable on ARM64**. Check this on real ARM hardware. The tray should detect
it and grey out the menu item.

---

## 3. The tray app (replaces `MenuBar/`)

### Technology choice

Use **C# WinForms targeting .NET Framework 4.8.** .NET Framework 4.8 ships with
every Windows 10 (1903+) and 11 machine, so there is **zero runtime to
install**. That's the same "no dependencies" promise the Mac app makes. The exe
is a few hundred KB.

- Build with the .NET SDK using an SDK-style project:
  `<TargetFramework>net48</TargetFramework>`, `<UseWindowsForms>true</UseWindowsForms>`.
  Add the `Microsoft.NETFramework.ReferenceAssemblies` package and it builds
  on any machine with the SDK. That's a build-time dependency only.
- JSON: use `System.Web.Script.Serialization.JavaScriptSerializer`
  (reference `System.Web.Extensions`). It's in the framework, so there's no
  NuGet package at runtime.
- Don't use .NET 8 for this. It would need either a ~60 MB self-contained
  publish or a runtime the user has to install.

*Alternative:* write the tray in Python with `pystray` + `tkinter`, so there's
only one language. The embeddable package has no tkinter, though, so you'd
switch to python-build-standalone. The UI will also look less native. Only
worth it if you really don't want C#.

### Swift → C# map

| macOS file | Windows file | Notes |
|---|---|---|
| `main.swift` | `Program.cs` | Parse `--replace <pid>`, `--redirect-service`, `--elevated <task>`, `--shutdown-all`. Take a named mutex `Local\WinDPI.Tray` for single instance. |
| `App.swift` | `TrayApp.cs` | `NotifyIcon` + `ContextMenuStrip`. Build the menu in `Opening` (= `menuNeedsUpdate`). Refresh on a 2-3 s `Timer`. **Never call the control channel on the UI thread**: use `Task.Run`, then `BeginInvoke`. |
| `Control.swift` | `Control.cs` | Read `control.json`, open TCP to `127.0.0.1:port`, send `{"cmd":…,"token":…}\n`, read one line, short timeouts. |
| `System.swift` → `SystemProxy` | `SystemProxy.cs` | §4 |
| `System.swift` → `LaunchAgent` | `Supervisor.cs` + `Autostart.cs` | §5 |
| `System.swift` → `Transparent` | `Redirect/*.cs` | §7 |
| `System.swift` → `runPrivileged` | `Elevation.cs` | Relaunch `WinDPI.exe --elevated <task>` with `Verb = "runas"` (UAC). Treat the `Win32Exception` with code 1223 as "user cancelled". |
| `Installer.swift` | `Installer.cs` | §6 |
| `Paths.swift` | `Paths.cs` | Install dir = exe dir. State = `%LOCALAPPDATA%\WinDPI`. |
| `Version.swift` | `Version.cs` | `build.ps1` reads `macdpi/__init__.py` and stamps it into `AssemblyInfo`. |

### Menu (same items as the Mac app)

Status header (Protected / Idle / Engine down) · connections, data, uptime ·
**Turn Protection On/Off** · **Strategy ▸** (auto + each strategy) ·
**Recent Sites ▸** · **Cover All Apps, Not Just Browsers** · **Excluded Sites…**
· **Test This Network** · Flush DNS · Forget Learned Strategies · Open Log ·
Start/Restart Engine · **Start at Login** · **Remove WinDPI…** · Quit.

Use three `.ico` files for the states (shield, outline shield, shield with
cross). Windows 11 hides new tray icons in the overflow area by default, so the
first-run dialog should say "drag the shield out of the ^ overflow to keep it
visible".

---

## 4. System proxy (replaces `networksetup`)

On Windows the browser proxy is **per-user and needs no admin**. That's easier
than macOS: no password at all for the basic mode.

Chrome, Edge, Brave, Opera, Vivaldi and most desktop apps read the **WinINET**
settings. Firefox follows them while it's on "Use system proxy settings" (its
default).

### Set it

```csharp
const string Key = @"Software\Microsoft\Windows\CurrentVersion\Internet Settings";
const string Bypass = "<local>;localhost;127.*;10.*;192.168.*;169.254.*;" +
    "172.16.*;172.17.*;172.18.*;172.19.*;172.20.*;172.21.*;172.22.*;172.23.*;" +
    "172.24.*;172.25.*;172.26.*;172.27.*;172.28.*;172.29.*;172.30.*;172.31.*";

[DllImport("wininet.dll", SetLastError = true)]
static extern bool InternetSetOption(IntPtr h, int option, IntPtr buf, int len);

static void Apply(bool on, int port) {
    using (var k = Registry.CurrentUser.OpenSubKey(Key, writable: true)) {
        if (on) {
            SaveOriginalOnce(k);                       // see below
            k.SetValue("ProxyServer", $"127.0.0.1:{port}");
            k.SetValue("ProxyOverride", Bypass);
            k.DeleteValue("AutoConfigURL", throwOnMissingValue: false);
            k.SetValue("ProxyEnable", 1, RegistryValueKind.DWord);
        } else {
            RestoreOriginal(k);
        }
    }
    InternetSetOption(IntPtr.Zero, 39 /*SETTINGS_CHANGED*/, IntPtr.Zero, 0);
    InternetSetOption(IntPtr.Zero, 37 /*REFRESH*/, IntPtr.Zero, 0);
}
```

The two `InternetSetOption` calls tell running browsers to re-read the
settings, so you don't have to restart anything.

If the Settings app (Settings → Network → Proxy) doesn't show the change on
some build, move to `INTERNET_OPTION_PER_CONNECTION_OPTION` (75) with an
`INTERNET_PER_CONN_OPTION_LIST`. That's the official API, and it also updates
the binary `Connections\DefaultConnectionSettings` blob. Start with the
registry approach, but check it in the Settings app.

### Restore, don't just switch off

The Mac version turns the proxy off. On Windows, **save the user's original
`ProxyEnable` / `ProxyServer` / `ProxyOverride` / `AutoConfigURL` /
`AutoDetect`** before the first change. Store them under
`HKCU\Software\WinDPI\SavedProxy`, and only if nothing is saved there yet.
Restore them exactly on Off and on Remove. A school or work PAC URL
(`AutoConfigURL`) that you delete and never put back is a support ticket for
someone.

### "Is it on?"

`ProxyEnable == 1 && ProxyServer` points at `127.0.0.1:<port>`. Also check
whether something else rewrote it: VPN clients, other proxy tools, and Windows
itself after some updates. If so, show "Idle (proxy changed by another app)".

### Known gaps

- **WinHTTP** is a separate proxy setting used by services, Windows Update and
  some launchers. It needs admin
  (`netsh winhttp import proxy source=ie`). Leave it alone by default and
  maybe offer it as an advanced option.
- **Store / UWP (AppContainer) apps can't reach 127.0.0.1** because of
  loopback isolation. With the system proxy on, some of them lose their
  connection altogether. The fix is per app, as admin:
  `CheckNetIsolation.exe LoopbackExempt -a -n=<PackageFamilyName>`. Put this
  in the docs, and possibly in an "Exempt Store apps…" menu item.
- **Chrome QUIC:** Chrome doesn't use QUIC through a configured proxy, so
  proxied browsing is all TCP and evasion applies.

---

## 5. Start at login, keep alive, self-heal (replaces launchd)

There's no launchd, so the tray app takes on that job.

1. **Autostart:** `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, value
   `WinDPI` = `"<install>\WinDPI.exe" --autostart`. No admin needed. This is
   the "Start at Login" toggle.
2. **The tray starts the engine** if `control.json`'s pid isn't alive or
   `status` doesn't answer. Use `ProcessStartInfo { UseShellExecute = false,
   CreateNoWindow = true }`. **Don't put it in a Job object with
   KILL_ON_JOB_CLOSE**: the engine has to keep running when the tray exits.
   Child processes on Windows survive their parent by default.
3. **The tray restarts the engine if it dies.** Copy launchd's behaviour: at
   most one restart per 10 s (`ThrottleInterval`), and stop trying after 5
   crashes in a minute. When it gives up, show the outage dialog.
4. **Outage handling:** port `checkForOutage()` as it is. Proxy on + engine
   silent for 3 refreshes → try a silent restart. Still silent at 6 → the
   dialog "WinDPI is not running" with **Restart Engine / Turn Protection Off
   / Ignore**.
5. **Self-heal a missing autostart:** write the `.configured` marker on
   first-run success (same as v1.2.1). If the marker exists but the `Run`
   value or the engine is missing, recreate it silently. Never fall back into
   first-run setup.
6. **Engine first, network later:** keep the v1.2.1 rule. Listeners and the
   control channel come up immediately at login, and `--wait-for-network`
   only runs in the background.
7. **Quit tray ≠ stop engine.** "Quit" leaves the engine and the proxy as
   they are, same as macOS. "Turn Protection Off" and "Remove" are the ways to
   stop.

Why not Task Scheduler or a service for the engine? A Windows service would
restart on its own and start before login. But it runs in session 0 as another
account, needs admin to install, and would put a network-facing process at a
higher privilege than it needs. A per-user process supervised by the tray
matches the Mac design. If you later want restarts that don't depend on the
tray, a per-user Task Scheduler task (logon trigger, "restart on failure") is
the next step up. Check whether your Windows edition lets a standard user
create a logon-triggered task before you rely on it.

*Optional fail-open mode:* instead of a fixed proxy, point WinINET at a PAC URL
the engine serves, `http://127.0.0.1:8881/proxy.pac` returning
`PROXY 127.0.0.1:8881; DIRECT`. If the engine is dead, browsers go direct
instead of failing. The trade-off: blocked sites quietly stay blocked and
nothing tells the user. Offer it as a setting if you like. Fail-closed plus
the outage dialog is the current macOS behaviour.

---

## 6. Install, update, uninstall (replaces build.sh's app bundle + DMG)

### Installer

Use **Inno Setup** (free) to build `WinDPI-Setup-<version>.exe`.

- **Install per machine** into `C:\Program Files\WinDPI` (one UAC prompt at
  install). It has to be Program Files: the redirect service (§7) runs as
  SYSTEM, and **a SYSTEM service whose exe sits in a user-writable folder (for
  example `%LOCALAPPDATA%`) lets any program on the machine replace that exe
  and gain SYSTEM**. Program Files is admin-only, which closes that hole.
- User state stays in `%LOCALAPPDATA%\WinDPI` (log, `learned.json`,
  `bypass.txt`, `control.json`, pycache).
- At the end of setup, run `WinDPI.exe` unelevated
  (`Flags: nowait postinstall runasoriginaluser`) so the first-run setup
  happens in the user's own context.
- Also ship a portable zip for people who don't want an installer. Cover All
  Apps would then need to copy itself to Program Files the first time it's
  turned on.

### First run (port `runFirstRunSetup`)

Welcome dialog → write the `Run` value → start the engine → wait for `status`
→ set the system proxy (no password on Windows) → "WinDPI is on" → the "drag
the shield out of the overflow" hint → `.configured` marker.

### Updates: Windows locks running files

On macOS you could overwrite the app while it ran and relaunch afterwards. On
Windows **you can't overwrite `pythonw.exe`, `python312.dll` or `WinDPI.exe`
while they're running.**

- At the start of setup (`[Code] InitializeSetup`), run
  `WinDPI.exe --shutdown-all`. It sends `shutdown` to the engine and the
  Minecraft route process, waits for their pids to exit (kills them after
  5 s), closes the tray, and stops the redirect service if it's running. Also
  set `CloseApplications=yes`, so Restart Manager catches anything left.
- After the files are copied, the post-install launch starts the new tray, and
  the tray starts the new engine.
- Run `bringUpToDate()` on every tray start, ported directly. If the `Run`
  value's path differs from this exe, rewrite it. If the engine's reported
  `version` differs from `Version.Current`, restart the engine.
- **Stale-copy guard** (the v1.1.1 lesson): store
  `HKCU\Software\WinDPI\{InstallDir, Version}`. A copy of `WinDPI.exe` started
  from anywhere else (Downloads, an old unzipped folder) **must not** point
  the `Run` value or the engine at itself if the registered copy is the same
  or newer. It should start the registered copy instead and exit. This is
  `Installer.newerCopyInCharge()` with a registry key instead of a plist.
- Single instance: if the mutex is held, send the running tray a "show
  yourself" message (a `WM_COPYDATA` or a second named pipe) and exit. For
  `--replace <pid>`, wait for that pid to exit before taking the mutex.

### Uninstall ("Remove WinDPI…" and the Inno uninstaller)

1. Restore the saved proxy settings (§4).
2. Stop and delete the redirect service, and remove the firewall rule (§7),
   with UAC.
3. Send the engine `shutdown`.
4. Delete the `Run` value and `HKCU\Software\WinDPI`.
5. Delete `%LOCALAPPDATA%\WinDPI`. Ask first: it holds the learned strategies.

Order matters: put the proxy back **before** stopping the engine, or browsers
are dead in between.

### Unsigned, like the Mac build

A code-signing certificate costs money, the same as the Apple Developer ID you
chose not to pay for. Without one:

- **SmartScreen** shows "Windows protected your PC". The user clicks
  **More info → Run anyway**, once per downloaded file. That's the Windows
  version of right-click → Open.
- Downloaded zips carry the **Mark of the Web**. Tell people to extract with
  Explorer (or `Unblock-File`) before running.
- **Windows 11 Smart App Control** (when it's in enforcement mode) blocks
  unsigned apps with **no override**. Users with SAC on can't run WinDPI
  without turning SAC off. Say so in the README.
- **Defender and other antivirus products often flag WinDivert-based tools**
  (GoodbyeDPI and zapret get "HackTool"/"RiskWare" labels). The browsers-only
  mode doesn't include WinDivert. Consider shipping WinDivert as a separate
  "Cover All Apps" component, so the basic install stays clean.

---

## 7. Cover All Apps (replaces pf)

There's no `pf` with `route-to` on Windows. Doing this properly in the kernel
(a WFP callout driver) needs a signed kernel driver, which needs an EV
certificate. **WinDivert** solves that: it's an open-source, **already-signed**
packet capture/inject driver. GoodbyeDPI and zapret are built on it.

### How the redirect works: "reflection"

The reference implementation is **`examples/streamdump/streamdump.c` in the
WinDivert repo**. Follow it closely. In short, for an outbound packet
`L:p → D:443`:

```
outbound  L:p → D:443         becomes inbound  D:p → L:8443   (swap addresses, dst port := listener)
outbound  L:8443 → D:p        becomes inbound  D:443 → L:p    (swap back, src port := 443)
```

Recalculate checksums and re-inject. The app thinks it's talking to `D:443`.
The engine's listener sees a connection **from `D`**, which means it learns the
**original destination IP** for free (`peername`). That's an improvement on
pf: the engine can route a ClientHello with no SNI to the IP it was really
meant for, instead of dropping it.

Consequences:

- The transparent listeners (8443/8480 and the Minecraft port) must bind
  **0.0.0.0** to accept these packets (they're addressed to the LAN IP, not
  loopback). The redirector must therefore **drop genuine inbound packets from
  the network** to those ports, and the installer should add a firewall rule
  that allows the engine to listen there. Otherwise Windows shows a firewall
  prompt, and your LAN neighbours can use your proxy.
- **Loop prevention** is the same as on macOS: the filter excludes source
  ports 40000-44999, which the engine's own upstream sockets use. **Read the
  band from the same constant as `netio.py`.**

### Filter (WinDivert filter language)

The engine generates it, like `pfgen.build_ruleset` today; call it
`divertgen.py`:

```
outbound and !loopback and tcp and
( tcp.DstPort == 443 or tcp.DstPort == 80 or tcp.DstPort == 25565
  or tcp.SrcPort == 8443 or tcp.SrcPort == 8480 or tcp.SrcPort == 8425 )
and (tcp.SrcPort < 40000 or tcp.SrcPort > 44999)
and !(ip.DstAddr >= 10.0.0.0 and ip.DstAddr <= 10.255.255.255)
and !(ip.DstAddr >= 172.16.0.0 and ip.DstAddr <= 172.31.255.255)
and !(ip.DstAddr >= 192.168.0.0 and ip.DstAddr <= 192.168.255.255)
and !(ip.DstAddr >= 169.254.0.0 and ip.DstAddr <= 169.254.255.255)
and !(ip.DstAddr >= 100.64.0.0 and ip.DstAddr <= 100.127.255.255)
and ...user bypass entries as ranges...
```

(Plus a second handle that drops `inbound and tcp.DstPort in {8443,8480,8425}`
from non-loopback sources.) `ALWAYS_BYPASS` and the bypass-file parsing from
`pfgen.py` carry over. Only the output format changes. The control command
`redirect_config` returns `{filter, listeners, src_band}`.

Start with **IPv4 only**, like the pf rules. IPv6 traffic on 443 escapes the
redirect, as it does on macOS. Either add IPv6 later or offer "block IPv6
TCP 443 while covering all apps" so Happy Eyeballs falls back to IPv4.
Optionally also drop outbound **UDP 443** in this mode, so QUIC apps fall
back to TCP, where evasion applies.

### Where the redirector runs

Run it as a **Windows service** inside the same `WinDPI.exe`
(`ServiceBase`, started with `--redirect-service`), calling `WinDivert.dll`
through P/Invoke: `WinDivertOpen`, `WinDivertRecvEx`,
`WinDivertHelperParsePacket`, `WinDivertHelperCalcChecksums`,
`WinDivertSendEx`. The packet rewriting is about 100 lines of C#.

- Write it in compiled code, not Python. Every packet of every redirected
  connection passes through it twice. Python with `pydivert` is fine for a
  **prototype** (and quick to try), but too slow and CPU-hungry to ship.
- Installed by the elevated step (UAC once). This is the Windows version of
  the pf LaunchDaemon:
  `sc create WinDPIRedirect binPath= "\"C:\Program Files\WinDPI\WinDPI.exe\" --redirect-service" start= auto`
  and `sc failure WinDPIRedirect reset= 60 actions= restart/5000`.
- **Configuration comes only from an admin-only file**,
  `%ProgramData%\WinDPI\redirect.json`, written by the elevated step. The
  service never takes instructions from the (unprivileged) engine or tray.
  Editing exclusions while it's on means another UAC prompt, exactly like the
  macOS password prompt for `reloadRules`.
- **Engine liveness watchdog.** This is the fail-safe that pf never had. Every
  second the service tries to connect to the engine's TLS listener. If that
  fails 3 times, it **closes its WinDivert handle**, and traffic flows
  normally (unprotected, but working). It reopens the handle when the engine
  answers again. If the service itself crashes, the handle closes with it,
  which is also fail-open.

### Turning it on (port `enableAllApps`, same order)

1. Restart the engine with `--transparent` and wait for `status`.
2. Ask the engine for `redirect_config`.
3. Elevated step (UAC): write `redirect.json`, add the firewall rule, install
   or start the service.
4. **Self-test:** control `selftest`, unchanged. The engine sends the
   `macdpi-selftest.invalid` ClientHello from a normal port to `1.1.1.1:443`
   and expects to receive it on its own listener.
5. If the self-test fails: stop the service, revert the engine, and tell the
   user nothing was changed. This is the same rollback as on macOS. The most
   common cause on Windows is another WinDivert user (GoodbyeDPI, zapret, some
   VPNs and game "boosters") already running. Detect `WinDivert` already being
   loaded by a process other than yours, and say so.

### Bonus: things only Windows can do

With WinDivert you can inject raw packets, so the strategies the DPI findings
list as "impossible from an unprivileged process" become possible in the
service. These are **fake packets** with a low TTL, a bad checksum or a wrong
sequence number, which a DPI box reads and the server never accepts. They're
GoodbyeDPI's and zapret's strongest options against reassembling middleboxes.
They'd have to live in the privileged redirector, which breaks rule 1, so
leave this for later and measure it with a per-strategy survey on both
networks before you ship it.

---

## 8. Minecraft

- **The route agent jar runs as it is.** It hooks Netty, not anything
  OS-specific. Install it to `%LOCALAPPDATA%\WinDPI\macdpi-mcroute.jar` and have
  the user add this to the launcher's JVM arguments
  (Installations → Edit → More options):
  `-javaagent:C:\Users\<you>\AppData\Local\WinDPI\macdpi-mcroute.jar`
  Backslashes are fine. Put the path in quotes if the user name has a space.
- Keep the separate `--minecraft-route` engine process on **8426**. The tray
  supervises it the same way it supervises the main engine. The 8425/8426
  clash came from pf's `rdr`; WinDivert reflection doesn't have it, but there's
  no reason to change the port.
- Under Cover All Apps, TCP 25565 is reflected like 443 (§7). The engine's
  `handle_transparent_raw` reads the server name from the Minecraft handshake,
  as it does today.
- The honest limit hasn't changed: this only disguises the handshake, not a
  sustained stream.

---

## 9. CLI (`dpictl` → `dpictl.ps1`)

Same verbs, in PowerShell: `start`, `stop`, `on`, `off`, `status`,
`test [hosts]`, `install` (Run key), `uninstall`, `log`
(`Get-Content -Wait -Tail 50`). `on`/`off` call the same registry and
`InternetSetOption` logic. The easiest way is for the script to call
`WinDPI.exe --proxy on|off`, so the logic lives in one place. Users may need
`powershell -ExecutionPolicy Bypass -File dpictl.ps1` for an unsigned script.

---

## 10. Testing recipe (Windows version of the Mac one)

- **Isolated engine beside an installed one:**
  ```powershell
  $env:LOCALAPPDATA = "$env:TEMP\windpi-scratch"
  & "C:\Program Files\WinDPI\python\python.exe" -X utf8 -m macdpi `
      --port 8899 --transparent --tproxy-tls-port 8543 --tproxy-http-port 8580 -v
  ```
  Changing `LOCALAPPDATA` for that one shell gives it its own state and its
  own `control.json`. `--transparent` keeps its upstream sockets in the
  reserved band, as on macOS. Stop it before running the installer, or the
  installer can't replace `python312.dll`.
- **Raw strategy probes:** unchanged (`Upstream.connect(..., reserve_source_port=True)`
  + `TLSStream`). Classify failures the same way: cert error = forged,
  timeout = dropped, reset = corrupted.
- **Sign-in proof without credentials:** same Xbox/Minecraft endpoints. The
  launcher log on Windows is
  `%APPDATA%\.minecraft\launcher_log.txt`, or under the Store launcher's
  package folder for the Microsoft Store launcher.
- **What the installed engine did:** `WinDPI.exe --status` (prints the control
  `status` JSON), or read `%LOCALAPPDATA%\WinDPI\macdpi.log`.
- **First check for "it stopped working"** (the stale-copy lesson):
  ```powershell
  Get-ItemProperty HKCU:\Software\Microsoft\Windows\CurrentVersion\Run | Select WinDPI
  Get-ItemProperty HKCU:\Software\WinDPI
  Get-CimInstance Win32_Process -Filter "name='pythonw.exe'" | Select ProcessId, CommandLine
  ```
- Test on **both** of your networks before reordering anything. The
  per-network results in the DPI findings are about the network, not the OS,
  so they should hold. Windows' TCP stack segments differently from macOS,
  though, so re-run the full strategy survey once.

---

## 11. Build script (`windows/build.ps1`)

1. Read the version from `macdpi/__init__.py`.
2. Download the embeddable Python zips (amd64 and arm64) into a cache and
   **check their SHA-256** against the values python.org publishes. Extract
   them and patch `python312._pth`.
3. Copy `macdpi/` (no `__pycache__`, no tests) and `cacert.pem`.
4. `dotnet build WinTray -c Release` (net48, AnyCPU) with the version stamped in.
5. Copy `WinDivert.dll` / `WinDivert64.sys` from the **official WinDivert
   release zip**, again checked by SHA-256.
6. Build the route agent jar (`mcroute-agent/build.sh` works from Git Bash or
   WSL, or port its 40 lines to PowerShell).
7. `iscc windows\WinDPI.iss` → `WinDPI-Setup-<v>.exe`, and zip the folder into
   `WinDPI-<v>-portable.zip`.
8. Smoke test: run `python\python.exe -X utf8 -m macdpi --help` from the
   output folder, and confirm that nothing writes into the output folder
   (no `__pycache__`).

Release the same way as the Mac app: tag `v<version>` and attach the setup exe
and the portable zip. Keep the Windows and macOS versions in step, since
they share the engine.

---

## 12. Suggested order of work

| # | Milestone | Done when |
|---|---|---|
| 1 | Engine runs on Windows (§1) | All checks in 1.11 pass, Wireshark confirms each strategy's segmentation |
| 2 | Bundled runtime (§2) | The engine runs from the build folder on a clean Windows VM with no Python installed |
| 3 | Tray + control + system proxy (§3-4) | Browsing works in Chrome, Edge and Firefox; Off restores the original settings exactly |
| 4 | Autostart, supervision, outage handling (§5) | Reboot → protected within seconds of login. Kill the engine → back within ~10 s. Quit the tray → browsing still works |
| 5 | Installer, update, uninstall (§6) | Install v1 → install v2 over it while it runs → v2 engine running with no reboot. An old copy run from Downloads doesn't take over. Uninstall leaves no trace and the proxy restored |
| 6 | Minecraft (§8) | Agent → 8426 → real server status, as proven on the Mac |
| 7 | Cover All Apps (§7) | Prototype with pydivert, then the C# service. Self-test passes; killing the engine fails open within ~3 s; another WinDivert tool running is detected and the change rolled back |
| 8 | Release | Setup exe + portable zip on GitHub, README with the SmartScreen / Smart App Control / antivirus notes |

---

## Appendix: macOS lessons mapped to Windows

| Lesson learned on macOS | Windows version |
|---|---|
| A stale old copy re-pointed the launch jobs at itself | Stale copy rewrites the `Run` value. Guard with the `HKCU\Software\WinDPI` version check (§6) |
| A second launch of a running app did nothing | Named mutex + `--replace <pid>`; the installer runs `--shutdown-all` because files are locked |
| The engine outlived an app replacement | `bringUpToDate()` on every tray start, comparing the engine's version |
| `__pycache__` written into the bundle broke signing | Program Files isn't writable anyway; keep `-X pycache_prefix` |
| launchd's 256-descriptor limit | Proactor loop, **not** the selector loop (512-socket `select()` limit) |
| Engine waited for the network before listening | Same rule: listen and serve the control channel immediately |
| Missing launchd job forced a reinstall | `.configured` marker + recreate the `Run` value silently |
| Translocation / quarantine | Mark of the Web, SmartScreen, Smart App Control |
| Reserved source ports clashed (`EADDRINUSE`) | Also Hyper-V/WSL excluded port ranges (`WSAEACCES` 10013) |
| A cached strategy "succeeds" against a forging middlebox | Unchanged, plus the lazy Windows root store and antivirus HTTPS scanning (§1.9-1.10) |
| A partial first read of the ClientHello | Unchanged (`_complete_record`) |
| `SO_REUSEADDR` was harmless | On Windows it lets two engines share a port. Use `SO_EXCLUSIVEADDRUSE` |
| launchd captured stderr | `pythonw` has no stderr. The engine writes its own log |
