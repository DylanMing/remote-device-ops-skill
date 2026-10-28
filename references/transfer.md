# Move Files In and Out

Copy the minimum. Every extra file is a version skew, a permission problem, or a secret you did not mean to ship.

## Choose the transport

| Situation | Use |
| --- | --- |
| A few small files to a POSIX target | `scp` |
| Many files, repeated syncs, resumable | `rsync -avz --partial --progress` |
| Target has no `rsync` (common on Windows, thin embedded images) | `tar` over the SSH pipe, or `scp -r` |
| Preserve permissions, symlinks, ownership hints | `tar` pipeline |
| Destination needs root | copy to `/tmp`, then `sudo mv` on the remote |

```bash
# POSIX push
scp ./<tool> <host>:<workspace>/bin/<tool>
rsync -avz --exclude '.git' --exclude '.venv' ./src/ <host>:<workspace>/src/

# POSIX pull
scp <host>:<workspace>/logs/<run>.log ./logs/
rsync -avz <host>:<workspace>/logs/ ./logs/

# tar pipe: fastest for trees, preserves modes, no rsync needed
tar czf - -C ./payload . | ssh <host> 'mkdir -p <workspace>/payload && tar xzf - -C <workspace>/payload'
ssh <host> 'tar czf - -C <workspace>/logs .' > ./logs.tgz
```

`rsync` over SSH uses `ssh` for auth; pass a key with `-e 'ssh -i ~/.ssh/id_ed25519'`. On Windows targets, check `Get-Command rsync` first — if absent, tar or `scp` avoids installing anything.

## Exclude the usual junk

`--exclude` (rsync) or `tar --exclude`:

```text
.git  .gitignore  .venv  venv  node_modules  __pycache__  *.pyc
build  dist  target  .pytest_cache  .mypy_cache  .DS_Store  Thumbs.db
*.log  tmp/  .env  secrets/  *.pem  *.key  *.p12
```

Two of those are security rules, not hygiene: **private keys and `.env` files never travel** unless the task explicitly requires a credential file, in which case copy it alone, lock down permissions, and report it.

## Windows path quirks

- `scp` splits on the first colon: `host:C:/work/file` is fine, but a local path containing a colon or a remote path with spaces needs quoting. Quote the remote side as a unit: `scp ./f <host>:'C:/Program Files/x/f'`.
- Modern OpenSSH uses the SFTP subsystem for `scp`; on a target where the `sftp-server` subsystem is missing, fall back with `scp -O` (legacy protocol) or use `tar`/`ssh 'cat > file'`.
- Prefer forward slashes and drive-letter absolute paths (`D:/work/bin/tool.exe`). Backslash paths get eaten by shell escaping.
- Copying an `.exe` does not make it runnable: check the ACL and mark the file if SmartScreen/zone metadata is a factor on the target.

## Verify the copy landed intact

```bash
sha256sum ./<file>                      # local
ssh <host> 'sha256sum <workspace>/bin/<file>'   # remote
```

Windows: `Get-FileHash <path> -Algorithm SHA256`, or `certutil -hashfile <path> SHA256`.

Compare sizes when a hash tool is unavailable; a truncated `scp` (dropped connection) is a classic silent failure.

## Compatibility, before you run it

A copied binary that cannot start is a compatibility bug, never a "the tool is broken" bug.

```bash
ssh <host> 'file <workspace>/bin/<tool>'
ssh <host> 'ldd <workspace>/bin/<tool>'        # glibc: report missing libs
ssh <host> 'uname -m'                          # compare with the build target
```

Checklist before first execution: OS family, architecture, libc (glibc vs musl), dynamic libraries present, interpreter version for scripts, executable bit (`chmod +x` on POSIX — `scp` does not always preserve it), and whether the SSH user can read/execute the path.

`Exec format error` → wrong architecture or a script without a valid shebang. `error while loading shared libraries` → missing runtime libs or wrong libc. `Permission denied` → missing execute bit or a `noexec` mount.

## Electing what to bring back

Pull only what proves or explains the run:

- The dated log for this run (not the whole `logs/` directory).
- Result artifacts: metrics JSON/CSV, captured media, generated reports, crash dumps.
- Environment fingerprint if it changed.

Never pull credentials, large caches, or the whole workspace "just in case" — it makes the report unreadable and the transfer slow.

## Cleanup

```bash
ssh <host> 'rm -rf <workspace>/tmp/<run-id>'      # your own temporary tree only
```

Remove temporary upload directories when done, and say what you left behind (binaries, venvs, logs) so the device owner can decide. Deleting anything outside your own `<workspace>` requires explicit approval.
