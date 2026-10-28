# Windows Targets over OpenSSH

Windows OpenSSH works, but it differs from a POSIX target in four places: where keys live, what `PATH` the session sees, how quoting survives the shell, and whether a process outlives the session. Handle those and it behaves.

## Key placement: two possible files

Windows `sshd` chooses based on group membership.

| Account | `authorized_keys` location |
| --- | --- |
| Regular user | `C:\Users\<User>\.ssh\authorized_keys` |
| Member of `BUILTIN\Administrators` (with the default `Match Group administrators` block) | `C:\ProgramData\ssh\administrators_authorized_keys` |

Check which applies before writing anything:

```powershell
whoami /groups | findstr /i administrators
Get-Content C:\ProgramData\ssh\sshd_config |
  Select-String "AuthorizedKeysFile|Match Group administrators|PubkeyAuthentication"
```

Write the key with **ASCII or UTF-8 without BOM** — Windows PowerShell 5.1's `Out-File`/`>` default to UTF-16LE, and a UTF-16 `authorized_keys` is silently rejected:

```powershell
$pub  = 'ssh-ed25519 AAAA... comment'
$path = "$env:USERPROFILE\.ssh\authorized_keys"
New-Item -ItemType Directory -Force (Split-Path $path) | Out-Null
[System.IO.File]::WriteAllText($path, $pub.Trim() + "`n", [System.Text.Encoding]::ASCII)
```

Then fix the ACLs — OpenSSH refuses files writable by others, and the error looks like a key failure:

```powershell
$dir  = "$env:USERPROFILE\.ssh"
$path = "$dir\authorized_keys"
$me   = "$env:USERDOMAIN\$env:USERNAME"
icacls $dir  /inheritance:r
icacls $dir  /grant:r "${me}:(F)" "SYSTEM:(F)"
icacls $path /inheritance:r
icacls $path /grant:r "${me}:(F)" "SYSTEM:(F)"
```

For the administrators file:

```powershell
$path = 'C:\ProgramData\ssh\administrators_authorized_keys'
[System.IO.File]::WriteAllText($path, $pub.Trim() + "`n", [System.Text.Encoding]::ASCII)
icacls $path /inheritance:r
icacls $path /grant:r "Administrators:(F)" "SYSTEM:(F)"
icacls $path /setowner "Administrators"
Restart-Service sshd
```

Verify, do not assume:

```powershell
Get-Content $env:USERPROFILE\.ssh\authorized_keys
ssh-keygen -lf $env:USERPROFILE\.ssh\authorized_keys
icacls $env:USERPROFILE\.ssh\authorized_keys
```

The service is **`sshd`**, not `ssh`. `Restart-Service ssh` fails and misleads.

## PATH and install locations differ from the desktop

An SSH session typically sees a **shorter `PATH`** than a logged-in desktop, so tools installed per-user or via a GUI installer may be invisible:

```powershell
$env:Path
Get-Command python, py, ffmpeg, node -ErrorAction SilentlyContinue | Select-Object Name, Source
where.exe python
```

Prefer absolute paths (`D:\tool\bin\tool.exe`) or fix the `PATH` for the session explicitly:

```powershell
$env:Path = 'D:\tool\bin;' + $env:Path
```

A dependency visible in the GUI but not over SSH is a PATH/PATH-scope problem, not a missing install.

## Quoting: stop inlining, upload a script

Every layer (local shell → `ssh` → `cmd.exe`/PowerShell → the program) re-parses the string, and `cmd.exe` adds its own rules for `"`, `&`, `^`, `%`, and `!`. One-liners are fine for a single command; anything longer becomes unpredictable.

```powershell
# upload once, then invoke by full path
scp .\probe.ps1 <host>:'C:/work/scripts/probe.ps1'
ssh <host> 'powershell -NoProfile -ExecutionPolicy Bypass -File C:/work/scripts/probe.ps1'
```

`-NoProfile` avoids user profile side effects; `-ExecutionPolicy Bypass` avoids the policy block that otherwise silently kills the script. For a `.cmd`/`.bat`: `ssh <host> 'cmd.exe /c D:\work\scripts\run.cmd'`.

If you must inline, use the `-Command` form with a single-quoted *local* argument and double quotes inside:

```bash
ssh <host> 'powershell -NoProfile -Command "Get-Process | Where-Object { $_.CPU -gt 10 }"'
```

## Output encoding

Non-English Windows hosts can emit mojibake over SSH. Force UTF-8 for the session when text matters:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

or `chcp 65001` in a `cmd` context. When in doubt, write output to a file with `Out-File -Encoding utf8` and inspect the file instead of trusting the terminal.

## Processes, services, and lifetime

Inspect and stop by marker, as on Linux:

```powershell
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -like '*<marker>*' } |
  Select-Object ProcessId, ParentProcessId, Name, CommandLine | Format-List

Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -like '*<marker>*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

Services:

```powershell
Get-Service <name> | Select-Object Status, StartType
Restart-Service <name>          # needs an elevated token
sc.exe query <name>
Get-WinEvent -LogName System -MaxEvents 50 | Where-Object { $_.ProviderName -like '*<name>*' }
```

**Lifetime is the trap.** A process started from an SSH session may be terminated when the session ends. Options, weakest to strongest:

1. `Start-Process -WindowStyle Hidden -RedirectStandardOutput ...` — still tied to the session on some configurations.
2. `Register-ScheduledTask` — survives disconnect, survives logoff, has a real stop story.
3. A Windows service (`sc.exe create`) — the durable answer for a persistent daemon.

```powershell
$action = New-ScheduledTaskAction -Execute 'D:\work\bin\tool.exe' -Argument '--marker <tag>' -WorkingDirectory 'D:\work'
Register-ScheduledTask -TaskName '<tag>' -Action $action -Trigger (New-JobTrigger -Once -At (Get-Date)) -RunLevel Highest -Force
Start-ScheduledTask -TaskName '<tag>'
Get-ScheduledTaskInfo -TaskName '<tag>'
Stop-ScheduledTask  -TaskName '<tag>'
Unregister-ScheduledTask -TaskName '<tag>' -Confirm:$false
```

Always leave the task name in the report so the owner can remove it.

## Firewall and networking

```powershell
Get-NetFirewallRule -Enabled True -Direction Inbound | Select-Object -First 20 DisplayName, LocalPort, Action
Test-NetConnection -ComputerName 127.0.0.1 -Port <port>
Get-NetTCPConnection -LocalPort <port> -ErrorAction SilentlyContinue
```

Opening a port is a persistent, security-relevant change: confirm before `New-NetFirewallRule`, and remove the rule afterwards if it was only for the test.

## Line endings and text files

Windows tools often reject LF-only files (`.cmd`, `.bat`, some configs). Convert when a target demands it:

```powershell
(Get-Content -Raw .\script.cmd) -replace "`n", "`r`n" | Set-Content -NoNewline .\script.cmd
```

Conversely, when pulling logs or CSVs to the controller, normalize or parse tolerantly rather than assuming one convention.

## Quick pre-flight

```powershell
hostname; whoami; whoami /groups | findstr /i administrators
[System.Environment]::OSVersion.VersionString; $env:PROCESSOR_ARCHITECTURE
$PSVersionTable.PSVersion.ToString()
Get-Service sshd | Select-Object Status
```

If `sshd` is stopped, nothing else matters — and fixing it usually needs console access unless another remote channel exists.
