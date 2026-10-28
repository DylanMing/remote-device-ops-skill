# Remote Test Checklists

Use these as templates. Replace `<tool>`, `<producer>`, `<consumer>`, `<device>`, and `<test-marker>` with the concrete roles from the test specification. Do not assume a fixed program name.

## SSH and Host Identity

```bash
ssh -G <host> | sed -n '1,120p'
ssh -vvv -o BatchMode=yes -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 <host> echo ok
```

Check:

- HostName, Port, User, IdentityFile.
- Whether the server accepts the public key.
- Whether the error is network, host key, or authentication.

## Linux Remote Environment

```bash
ssh <host> 'set -eu; hostname; whoami; uname -a; uname -m; pwd; echo "$PATH"; ps -ef | head'
ssh <host> 'command -v python3 || true; command -v python || true; command -v uv || true; command -v pip || true'
ssh <host> 'command -v node || true; command -v npm || true; command -v java || true; command -v cargo || true; command -v docker || true'
ssh <host> 'ss -ltnp || true; ss -lunp || true'
```

Project-specific tool probe:

```bash
ssh <host> 'test -x <remote-path-to-tool> && <remote-path-to-tool> --help | head || true'
ssh <host> 'file <remote-path-to-tool> || true; ldd <remote-path-to-tool> || true'
```

## Windows Remote Environment

Prefer script files for complex checks. Minimal one-shot:

```bash
ssh <host> 'powershell -NoProfile -ExecutionPolicy Bypass -Command "hostname; whoami; Get-ComputerInfo | Select-Object WindowsProductName,OsVersion"'
```

Useful PowerShell snippet:

```powershell
$TestDir = 'D:\path\to\test'
$Runtime = Join-Path $TestDir '.venv\Scripts\python.exe'
$Tool = Join-Path $TestDir 'bin\your-tool.exe'

Write-Host "host=$(hostname)"
Write-Host "user=$(whoami)"
Write-Host "arch=$env:PROCESSOR_ARCHITECTURE"
Write-Host "path=$env:Path"
Write-Host "runtime=$Runtime exists=$(Test-Path $Runtime)"
Write-Host "tool=$Tool exists=$(Test-Path $Tool)"
where.exe python 2>$null
where.exe py 2>$null
where.exe ffmpeg 2>$null
if (Test-Path $Runtime) { & $Runtime --version }
if (Test-Path $Tool) { & $Tool --help }

Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -and $_.CommandLine.Contains('your-test-marker') } |
  Select-Object ProcessId,Name,CommandLine |
  Format-List
```

## Dependency Install Options

Prefer local-to-test installs:

```bash
ssh <host> 'cd <test-dir> && python3 -m venv .venv && .venv/bin/python -m pip install -r requirements.txt'
ssh <host> 'cd <test-dir> && uv venv .venv && uv pip install -r requirements.txt'
```

```powershell
cd D:\path\to\test
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
winget list
where.exe <tool>
```

For OS-level package installs, ask for approval first and record the exact package manager command.

## Build or Cross-Compile Decision

Use remote native build if the remote has the right toolchain:

```bash
ssh <host> 'cd <source-dir> && <build-command>'
```

Use controller/server build when the remote should only run artifacts:

```text
controller build -> package artifact -> scp/rsync -> remote smoke test
```

Before transfer, identify the target:

```bash
ssh <host> 'uname -s; uname -m'
ssh <host> 'python3 - <<EOF
import platform, sys
print(platform.platform())
print(platform.machine())
print(sys.version)
EOF'
```

```powershell
[System.Environment]::OSVersion.VersionString
$env:PROCESSOR_ARCHITECTURE
py -0p
```

## File Transfer

```bash
scp local_script.py <host>:'<remote-test-dir>/scripts/local_script.py'
scp local_tool.ps1 <host>:'D:/test-dir/scripts/local_tool.ps1'
scp local_binary <host>:'<remote-test-dir>/bin/local_binary'
ssh <host> 'powershell -NoProfile -ExecutionPolicy Bypass -File D:/test-dir/scripts/local_tool.ps1'
```

Before copying a directory, exclude:

- `.git`, `__pycache__`, `.pytest_cache`, `.venv`, `node_modules`, build caches.
- private keys, certificates, tokens, credentials.

## Process Lifecycle

Foreground debugging:

```bash
ssh <host> '<remote-test-dir>/scripts/run_test.sh'
ssh <host> 'cmd.exe /c D:\test-dir\scripts\run_test.cmd'
```

Windows process inspection:

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -and $_.CommandLine.Contains('test-marker') } |
  Select-Object ProcessId,ParentProcessId,Name,CommandLine |
  Format-List
```

Windows stop:

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -and $_.CommandLine.Contains('test-marker') } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

Linux process inspection:

```bash
ssh <host> "ps -ef | grep '<test-marker>' | grep -v grep || true"
ssh <host> "pkill -f '<test-marker>' || true"
```

## Optional Media Test Pattern

Use synthetic source first:

```text
testsrc -> encoder -> publisher -> relay -> subscriber -> viewer
```

Then real device:

```text
camera -> encoder -> publisher -> relay -> subscriber -> viewer
```

For Windows camera checks:

```powershell
D:\ffmpeg\bin\ffmpeg.exe -hide_banner -list_devices true -f dshow -i dummy
```

Watch for:

- Exact device name, including spaces.
- Quoting around names such as `"HD Camera"`.
- Device busy errors.
- Whether the hardware indicator turns on, if present.
