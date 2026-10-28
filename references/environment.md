# Profile the Device

Do this once per session, in one or two batch calls, and keep the output. Everything you later call "broken" gets judged against this baseline.

## Linux

```bash
ssh -o BatchMode=yes <host> 'set -u
echo "== identity =="; hostname; whoami; id; groups
echo "== os =="; (cat /etc/os-release 2>/dev/null | head -3); uname -srm
echo "== shell =="; echo "SHELL=$SHELL"; readlink -f /proc/$$/exe 2>/dev/null
echo "== cwd+path =="; pwd; echo "$PATH"
echo "== resources =="; nproc; free -m 2>/dev/null | head -2; df -h / /tmp 2>/dev/null
echo "== runtimes =="; for t in python3 python uv pip node npm java go cargo docker ffmpeg git systemctl; do \
  printf "  %-10s %s\n" "$t" "$(command -v $t 2>/dev/null || echo -)"; done
echo "== listening =="; (ss -ltnup 2>/dev/null || netstat -ltnup 2>/dev/null) | head -25
echo "== init =="; systemctl --version 2>/dev/null | head -1 || echo "no systemd"
echo "== privileges =="; sudo -n true 2>/dev/null && echo "sudo:passwordless" || echo "sudo:no-or-password"
echo "== hardware =="; ls /dev/video* /dev/ttyUSB* /dev/ttyACM* /dev/snd 2>/dev/null || echo "no obvious device nodes"
'
```

Notes:

- `command -v` shows what the **SSH shell** resolves, not what the GUI knows. Empty means "not on this PATH", not "not installed" — search the usual prefixes before installing anything.
- `sudo -n true` succeeds only for passwordless `sudo`. Success does not mean you may use it; it means you may do so without a prompt.
- Reading `/dev/video*` existence is not permission: check `id`, `groups`, and the node's mode.

## macOS

```bash
ssh <host> 'sw_vers; uname -m; sysctl -n hw.ncpu hw.memsize; df -h / ; launchctl list | head -20'
```

Look for Homebrew prefixes on both `/opt/homebrew` (arm64) and `/usr/local` (x86_64) — a tool can exist and still be invisible over SSH. `system_profiler SPUSBDataType` / `SPCameraDataType` lists attached hardware when it matters.

## Windows

Prefer uploading a `.ps1` and running it; inline quoting over SSH is fragile (see `references/windows.md`).

```powershell
$d = @{}
$d.host      = hostname
$d.user      = whoami
$d.os        = (Get-CimInstance Win32_OperatingSystem).Caption
$d.version   = [System.Environment]::OSVersion.VersionString
$d.arch      = $env:PROCESSOR_ARCHITECTURE
$d.psVersion = $PSVersionTable.PSVersion.ToString()
$d.path      = $env:Path
$d.cwd       = (Get-Location).Path
$d.runtimes  = 'python','py','node','npm','java','ffmpeg','git' | ForEach-Object {
                 $c = Get-Command $_ -ErrorAction SilentlyContinue
                 '{0} = {1}' -f $_, ($(if ($c) { $c.Source } else { '-' })) }
$d.listening = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
                 Select-Object -First 25 LocalAddress, LocalPort, OwningProcess
$d.disk      = Get-PSDrive -PSProvider FileSystem | Select-Object Name, Used, Free
$d.admin     = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
                 ).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
$d | Format-List
```

And the one-liner when you only need a sanity check:

```bash
ssh <host> 'powershell -NoProfile -ExecutionPolicy Bypass -Command "hostname; whoami; $env:PROCESSOR_ARCHITECTURE"'
```

Windows over SSH frequently has a **shorter `PATH`** than an interactive session, because machine and user `PATH` combine differently for the login. Full-path invocation is the norm, not the exception.

## Read the architecture correctly

| Reported | Means | Consequences |
| --- | --- | --- |
| `x86_64` / `AMD64` | 64-bit Intel | ordinary Linux/Windows binaries |
| `aarch64` / `ARM64` | 64-bit ARM | ARM builds; no x86 binaries without emulation |
| `armv7l` / `armhf` | 32-bit ARM | hard-float ABI matters for prebuilt binaries |
| `i686` / `x86` | 32-bit Intel | rare now; verify before deploying |
| `PROCESSOR_ARCHITECTURE=ARM64` with `x86` processes | ARM64 Windows running x64 under emulation | prefer native ARM64 builds |
| `aarch64` + `armv7l` mismatch | 64-bit kernel, 32-bit userland | compile for the userland ABI |

Also record the C library family (`ldd --version` → glibc/musl) and, for interpreted languages, the exact runtime version. A binary that starts but exits with `Exec format error` or `error while loading shared libraries` is an architecture/libc mismatch, not a broken program.

## Services, ports, and what owns them

```bash
ssh <host> 'systemctl list-units --type=service --state=running --no-pager | head -20'
ssh <host> 'ss -ltnp | grep <port> || echo "port free"'          # who owns the port
ssh <host> 'ps -ef | grep -F "<marker>" | grep -v grep || true'   # your own processes
```

On Windows: `Get-Service | Where-Object Status -eq 'Running'`, `Get-NetTCPConnection -LocalPort <port>`, `Get-CimInstance Win32_Process | Where-Object CommandLine -like '*<marker>*'`.

Before starting anything on a port, confirm it is free **and** that it is not on the forbidden list from step 0.

## Remote workspace layout

Keep the device tidy and the next run cheap:

```text
<workspace>/
  bin/        copied executables and project-specific tools
  scripts/    probe, run, stop, collect
  config/     task configs only
  data/       small input fixtures
  logs/       dated stdout/stderr and application logs
  packages/   wheels, installers, archives
```

Rules that pay for themselves: every long-running command carries a unique marker string; every run writes a timestamped log under `logs/`; every `run_*` has a matching `stop_*` that matches only that marker.

## Baseline summary to record

```text
host / user / ssh alias        os / version / arch / libc
shell / cwd / PATH highlights  runtimes and their exact paths
service manager                sudo/admin capability
listening ports (baseline)     hardware present, permissions
disk and memory headroom       workspace directory created?
```

Anything that changes later (a port disappears, a tool leaves `PATH`, disk fills) is a signal — but only if you wrote the baseline down.
