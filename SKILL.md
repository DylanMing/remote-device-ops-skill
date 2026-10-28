---
name: remote-device-test
description: Operate and verify remote devices over SSH from an automated agent. Use when connecting to a remote Linux, macOS, Windows, or embedded target; probing its OS, architecture, PATH, runtimes, ports, services, permissions, users, or hardware; running one-shot commands, scripts, or long-lived services; deploying binaries, packages, configs, or data; streaming logs and artifacts back to the controller; forwarding ports; and proving that a change actually took effect. Covers non-interactive authentication, quoting and exit-code semantics, background process lifetime, Windows OpenSSH specifics, file transfer, metric collection, and failure triage across access, environment, device, network, protocol, and lifecycle problems.
---

# Remote Device Ops

Drive a remote device from an automated agent over SSH: connect, inspect, change, verify, report. The same loop serves one-off commands, deployments, debugging, certification runs, and repeatable test campaigns.

Name things by **role**, never by product: `<target>`, `<service>`, `<tool>`, `<producer>`, `<consumer>`, `<relay>`, `<device-tool>`, `<analyzer>`. Bind each role to a concrete binary, unit, or endpoint only for the task at hand, so the workflow survives a change of project.

## Non-negotiables

1. **Prove the channel before touching anything.** `ssh <host> echo ok` comes first. If it fails, it is an access problem — never start debugging application code.
2. **Assume no TTY.** The agent cannot answer a password prompt, a host-key question, or a `sudo` password. Every step must be non-interactive, or fail fast and loudly.
3. **Probe inside the context that will do the work.** SSH sessions routinely differ from the GUI or scheduled task in `PATH`, environment, working directory, privileges, and locale. Never assume a GUI-installed tool is visible over SSH.
4. **Escalate in small steps.** One-shot probe → minimal command → smoke run → real work. Never open with the real run.
5. **Make it stoppable and repeatable.** Before anything long-lived starts, know the host, exact command, working directory, PID or unit name, log path, and stop command.
6. **Verify from both sides.** Remote evidence (exit code, process, listening port, log lines) *and* the outcome that matters (payload received, file emitted, metric reached, output visible). Exit code 0 is not proof.
7. **Nothing privileged, destructive, or persistent without consent.** Package installs, service restarts, firewall changes, reboots, `rm -rf`, partition/firmware work: state the scope, prefer a dry run, then act.

## Workflow

### 0. Frame the task

Write this down before running anything; it is what you will report against later.

```yaml
goal: <what must be true when this is done>
host: <ssh alias | user@addr:port>        # jump host, key path, passphrase source
device: <linux | macos | windows | embedded>   arch: <x86_64 | aarch64 | armv7l | other>
allowed: [read, run, copy-in, copy-out, install, restart-service, sudo, reboot, forward-port]
forbidden: [<ports | services | files | hardware to leave alone>]
workspace: <remote directory for bin, scripts, config, data, logs>
done_when: <one observable acceptance check>
```

If `allowed` is unknown, assume read-only and ask. If `forbidden` is unknown, ask before touching ports, services, or hardware.

### 1. Establish the channel

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 <host> 'echo ok; whoami; hostname'
```

Then pin down what is actually in effect — `ssh -G <host>` prints the resolved config (HostName, Port, User, IdentityFile, ProxyJump). Persist a working invocation into `~/.ssh/config` so later steps are one word.

Failure here is access: read `references/connect.md`, fix auth, and stop. Do not work around a broken channel with saved passwords, `sshpass` on a shared box, or disabled host-key checking.

### 2. Profile the device

One batch probe beats six round trips. Collect OS, architecture, shell, `PATH`, working directory, runtimes, package managers, listening ports, service manager, `sudo` capability, free disk and memory, and attached hardware in a single call.

Exact probes for Linux, macOS, and Windows are in `references/environment.md`. Cache the result in the report; it is the baseline every later anomaly is judged against.

### 3. Change something

Pick the smallest execution shape that works:

- **Inline one-liner** — single command, no branching, easy to read back in the log.
- **Uploaded script** — anything with quoting, loops, multiple steps, or platform-specific syntax. Prefer a real file over escape gymnastics.
- **Service or unit** — anything that must outlive the SSH session.

Rules: absolute paths for project-specific executables unless `PATH` was verified in this session; `set -euo pipefail` in scripts; a unique **marker string** in every long-running command line so it can be found and stopped later; explicit working directory.

Semantics that bite — quoting layers, `stdin` consumption, exit-code propagation (`255` means SSH itself failed, not your command), timeouts, `sudo`, background lifetime — are all in `references/execute.md`.

### 4. Move artifacts deliberately

Copy in only what the task needs; pull back only logs and results. Exclude `.git`, `.venv`, `node_modules`, caches, build outputs, keys, certificates, and tokens. Verify integrity and target compatibility (OS, architecture, libc, runtime version) before executing a copied binary.

`references/transfer.md` covers scp/rsync/tar-over-ssh selection, exclusion rules, checksums, and Windows path quirks.

### 5. Verify

Walk the evidence ladder, do not stop at the first rung:

1. Process alive / unit active / port listening.
2. Log lines from *this* run, with a timestamp after your start time.
3. The business outcome: bytes, messages, frames, files, rows, exit codes, rendered output.
4. The stated metric, measured over a window (see `references/verify.md`).
5. Clean stop: no orphaned processes, no leaked ports, no leftover temporary files.

Beware the classic false positives: exit 0 from a launcher whose child died; a log file from the previous run; a counter that only ever accumulated; a decoder that reports frames while nothing is visible.

### 6. Close out

Stop what you started (or hand over the exact stop command), clean temporary artifacts, and report:

```text
Device:     <host> / <user> / <os-arch>   session: <ssh command used>
Workspace:  <remote dirs touched>
Started:    <command>  -> pid/unit <id>  -> logs <paths>
Stopped:    <command or "left running, stop with: ...">
Evidence:   <exit code, key log lines, port, payload count, metric values>
Conclusion: works | works with caveats | failed  — <one line why>
Risks:      <what is fragile, what was not verified, what needs the owner>
```

## Routing

| You need to… | Read |
| --- | --- |
| Fix auth, host keys, jump hosts, keepalive, connection reuse | `references/connect.md` |
| Know OS, arch, PATH, runtimes, ports, services, hardware, permissions | `references/environment.md` |
| Quote correctly, pass stdin, get exit codes, time out, `sudo`, background a service | `references/execute.md` |
| Move files in/out, exclude junk, verify integrity, check arch/libc | `references/transfer.md` |
| Choose metrics, collect windowed rates, prove it works, write the report | `references/verify.md` |
| Something failed and you must classify it before changing anything | `references/diagnose.md` |
| Target is Windows: keys, encoding, services, quoting, task scheduler | `references/windows.md` |

## Hard rules

- **Never** put a password, token, or private key on a command line; it lands in `ps`, shell history, and logs. Use keys, agent forwarding, or a file with `600`/ACL-restricted permissions.
- **Never** disable host-key verification to "make it work"; fix or trust the key deliberately.
- **Never** leave a hidden remote process running without reporting it and its stop command.
- `ssh` exits with the remote command's status, and **255** when SSH itself failed. Distinguish the two before blaming the remote program.
- Non-interactive SSH may skip `.bashrc`/profile; when the login environment matters, use `ssh <host> 'bash -lc "<cmd>"'`. Prefer absolute paths over guessing.
- Redirect all three descriptors (`< /dev/null > log 2>&1`) for detached processes, otherwise the SSH session can hang or the child can be killed.
- One variable per change. If the result is unexpected, revert to the last known-good state before trying the next idea.

## Worked shapes

**Deploy a binary and keep it running**

```bash
scp ./<tool> <host>:<workspace>/bin/<tool> && \
ssh <host> '<workspace>/bin/<tool> --version' && \
ssh <host> 'cd <workspace> && setsid nohup bin/<tool> --marker <tag> \
  < /dev/null > logs/<tool>.log 2>&1 & echo started $!' && \
ssh <host> "ps -ef | grep -F '<tag>' | grep -v grep"
```

**Reach a remote-only port from the controller**

```bash
ssh -N -o ExitOnForwardFailure=yes -L 127.0.0.1:9000:127.0.0.1:9000 <host>
```

**Stream a media pipeline end to end** (roles are illustrative, not required)

```text
synthetic source -> encoder -> publisher -> relay -> subscriber -> viewer
```

Run the synthetic source first, confirm the chain forwards frames and the viewer shows output, and only then swap in the real device (camera, sensor, capture card). A pipeline that is healthy in counters but black on screen is a failure, not a pass.

**Repeatable campaign** — turn the framed task from step 0 into a stored spec plus `run_*` / `stop_*` scripts in `<workspace>`, each emitting the marker string and a dated log path, so the next run is one command and the result is comparable.
