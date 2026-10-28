# Run Commands Correctly

Most "the remote command failed" incidents are quoting, `stdin`, environment, or process-lifetime mistakes. Handle those four and remote execution becomes boring.

## Know how many layers your text passes through

```
agent's local shell  →  ssh argument  →  remote login shell  →  the program
```

Every layer expands variables and eats quotes. The reliable default:

- Wrap the **entire remote command** in **single quotes locally**: `ssh host 'cmd --flag="value"'`.
- Inside, use **double quotes only where the remote shell must expand something** (`"$VAR"`).
- When the remote command needs a literal single quote, a loop, or two+ lines: **stop inlining and upload a script**. Escape chains like `'\''` are where agents lose an hour.

```bash
# correct: remote expands $HOME, local does not
ssh host 'ls -l "$HOME/data"'

# wrong: local expands $HOME before ssh ever sees it
ssh host "ls -l $HOME/data"

# avoid the escape treadmill
cat > /tmp/step.sh <<'EOF'
for f in "$HOME"/data/*.csv; do
  printf '%s\n' "$f"
done
EOF
scp /tmp/step.sh host:/tmp/step.sh && ssh host 'bash /tmp/step.sh'
```

## Feed scripts through stdin instead of escaping them

No temp file, no quoting, delimiters stay literal when the heredoc marker is quoted:

```bash
ssh host bash -s <<'EOF'
set -euo pipefail
hostname
for f in /data/*.bin; do sha256sum "$f"; done
EOF
```

Equivalent, when the script already exists: `ssh host 'bash -s' < ./probe.sh`.

## `stdin` is a shared, easily stolen resource

`ssh` forwards your stdin to the remote command unless told otherwise. Two consequences:

```bash
# BROKEN: ssh swallows the rest of the file/loop, only the first host is processed
while read -r h; do ssh "$h" 'uptime'; done < hosts.txt

# correct
while read -r h; do ssh -n "$h" 'uptime'; done < hosts.txt

# or refuse stdin explicitly
ssh host 'some-tool --batch' </dev/null
```

Rule of thumb: **`-n` for every ssh call inside a loop or pipeline; `</dev/null` for anything that must never wait for input.**

## Exit codes: distinguish "program failed" from "ssh failed"

`ssh` returns the remote command's exit status, and **255 when SSH itself could not run the command** (connection, auth, transport). Never read 255 as an application error.

```bash
ssh host 'exit 3'; echo $?        # 3  -> remote program's status
ssh -o ConnectTimeout=3 bad.host true; echo $?   # 255 -> ssh-level failure
```

Notes:

- `ssh host 'a; b'` returns the status of the **last** command. Use `a && b` or an explicit script with `set -e` when earlier steps matter.
- `set -euo pipefail` inside uploaded scripts turns silent partial failures into loud ones.
- Capture stderr too: `ssh host 'cmd' 2>&1 | tee run.log`, or redirect to a file on the remote and pull it back.
- Set `LC_ALL=C` when you intend to parse tool output, so locale-specific decimals and translated messages do not break your regex.

## Timeouts at three levels

```bash
# 1. connection-level
ssh -o ConnectTimeout=10 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <host> ...

# 2. command-level on the remote (GNU coreutils)
ssh <host> 'timeout --signal=TERM 120 <tool> --args'

# 3. controller-level: run the ssh call as a background job and kill it
```

The third is the only one that survives a hung transport. Whatever you do, **never leave an unbounded call running**: a test that can hang forever is a test that has already failed.

Windows has no `timeout.exe` equivalent for processes; wrap with `Start-Process` + `Wait-Process -Timeout`, or use PowerShell jobs.

## Environment variables and login vs non-login shells

Non-interactive SSH runs a shell that may skip `.bashrc`/`.profile`, so `PATH`, aliases, virtualenvs, and vendor SDK variables can all be missing relative to the GUI. Two fixes:

```bash
ssh <host> 'bash -lc "<cmd>"'          # get the login environment
ssh <host> '/opt/vendor/bin/<tool> --version'   # or just use absolute paths
```

Do not rely on `SendEnv` unless it is configured on both sides (`AcceptEnv` in `sshd_config`), and never depend on an interactive alias existing.

For a one-off override, inline assignment is enough and does not leak into other calls: `ssh <host> 'FOO=1 BAR=2 <tool>'`.

## Privilege escalation

```bash
ssh <host> 'sudo -n true'          # does passwordless sudo work?
ssh <host> 'sudo -n systemctl restart <service>'
```

- The agent cannot type a `sudo` password. `echo pw | sudo -S` puts the password into `ps` and history on both ends — do not.
- If a specific `NOPASSWD` rule is acceptable, that is the clean solution; otherwise ask a human to perform the privileged step.
- On Windows, "privileged" means the SSH session's token: check `IsInRole(Administrator)` rather than assuming. An elevated GUI session does not imply an elevated SSH session.

## Long-lived processes

A bare `&` is not enough: `sshd` keeps the channel open while any process still holds the stdout pipe, and on some platforms killing the session kills the child.

```bash
ssh <host> 'cd <workspace> && \
  setsid nohup bin/<tool> --config config/<file> </dev/null >logs/<tool>.log 2>&1 & \
  echo "pid=$!"'
```

The four required pieces: `setsid` (own session, survives disconnect), `nohup` (ignore SIGHUP), `< /dev/null` (no stdin), `>log 2>&1` (never hold the terminal pipe).

Better on a systemd host — the unit owns restart policy, logging, and stop semantics:

```bash
ssh <host> 'systemd-run --user --unit=<name> --collect bin/<tool> --args'
ssh <host> 'systemctl --user status <name>; journalctl --user -u <name> -n 50 --no-pager'
ssh <host> 'systemctl --user stop <name>'
```

On Windows, an SSH-launched child can die with the session. For anything that must outlive it, register a Scheduled Task (`Register-ScheduledTask`) or a service rather than backgrounding a process.

## Make processes findable and killable

Give every long-running command a unique marker and record the PID:

```bash
ssh <host> 'ps -ef | grep -F "<marker>" | grep -v grep'
ssh <host> 'pkill -f "<marker>"'          # narrow with -u/-f, never a bare pkill <name>
```

Windows equivalent: `Get-CimInstance Win32_Process | Where-Object CommandLine -like '*<marker>*'` and `Stop-Process -Id <pid>`.

A marker that matches only your own run is what makes "stop" safe on a shared device.

## Sequencing

- `cd` does not persist between ssh calls; put `cd <dir> && ...` in the same invocation.
- One invocation, one logical step. If step 2 of five fails, a five-step one-liner leaves you unable to tell where it stopped.
- Prefer idempotent commands; when overwriting state, read the current value first so it can be restored.
- Log to a file for anything longer than a screenful — SSH output gets truncated, files do not.
