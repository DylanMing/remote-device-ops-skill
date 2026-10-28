# Windows OpenSSH Reference

Use this when SSH to a Windows device fails, asks for a password, or behaves differently inside Codex than in a user terminal.

## Non-Interactive Login

Codex cannot type an SSH password unless an interactive TTY is explicitly available. If `ssh <host>` works outside Codex but fails inside Codex with:

```text
Pseudo-terminal will not be allocated because stdin is not a terminal.
Permission denied (publickey,password,keyboard-interactive).
```

then public-key auth is not working and SSH fell back to password auth.

Test public-key login explicitly:

```bash
ssh -o PreferredAuthentications=publickey \
  -o IdentitiesOnly=yes \
  -i ~/.ssh/id_ed25519 \
  <host> echo ok
```

## Windows User Key Setup

For a normal Windows user, write the Linux public key to:

```text
C:\Users\<User>\.ssh\authorized_keys
```

PowerShell, run as that user:

```powershell
$pub = 'ssh-ed25519 ... comment'
$dir = "$env:USERPROFILE\.ssh"
$path = "$dir\authorized_keys"
$meRule = "$env:USERDOMAIN\$env:USERNAME" + ':(F)'

New-Item -ItemType Directory -Force $dir | Out-Null
[System.IO.File]::WriteAllText($path, $pub + [Environment]::NewLine, [System.Text.Encoding]::ASCII)
icacls $dir /inheritance:r
icacls $dir /grant:r $meRule "SYSTEM:(F)"
icacls $path /inheritance:r
icacls $path /grant:r $meRule "SYSTEM:(F)"
```

Verify:

```powershell
type $env:USERPROFILE\.ssh\authorized_keys
ssh-keygen -lf $env:USERPROFILE\.ssh\authorized_keys
icacls $env:USERPROFILE\.ssh\authorized_keys
```

## Administrator Group Behavior

If the user is in `BUILTIN\Administrators`, Windows OpenSSH may use:

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

Check:

```powershell
whoami /groups | findstr /i administrators
Get-Content C:\ProgramData\ssh\sshd_config |
  Select-String "AuthorizedKeysFile|Match Group administrators|PubkeyAuthentication"
```

If `Match Group administrators` is enabled, write the key to the administrators file:

```powershell
$pub = 'ssh-ed25519 ... comment'
$path = 'C:\ProgramData\ssh\administrators_authorized_keys'
[System.IO.File]::WriteAllText($path, $pub + [Environment]::NewLine, [System.Text.Encoding]::ASCII)
icacls $path /inheritance:r
icacls $path /grant:r "Administrators:(F)" "SYSTEM:(F)"
icacls $path /setowner "Administrators"
Restart-Service sshd
```

## Common Windows SSH Pitfalls

- `Restart-Service ssh` is wrong; the service is usually `sshd`.
- PowerShell 5.1 can write UTF-16 files; use `[System.IO.File]::WriteAllText(..., [System.Text.Encoding]::ASCII)`.
- SSH shell PATH often differs from the GUI PATH. Use full paths for `python.exe`, `ffmpeg.exe`, `moq-cli.exe`, etc.
- PowerShell quoting over SSH is fragile. Prefer uploading a `.ps1` or `.cmd` file and running it.
- Processes started from an SSH session may terminate when that SSH session closes. For debugging, keep the SSH command foreground. For production, use a real service manager.
