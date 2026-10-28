# Diagnose Before You Change Anything

Classify first. A wrong classification sends you rewriting application code for a firewalled port. Name the layer, then test that layer alone.

## Five classes

| Class | Owns | Typical evidence |
| --- | --- | --- |
| **Access** | ssh route, port, auth, host key, TTY, permissions of the login | `Connection refused`, `Permission denied (publickey)`, 255 exit |
| **Environment** | binary missing, wrong PATH, wrong interpreter/venv, version skew, arch/libc mismatch, locale | `command not found`, `Exec format error`, `No module named`, works interactively but not over SSH |
| **Device** | device node, driver, permissions, device busy, unsupported format, cabling | `/dev/video0` absent, `Device or resource busy`, `dshow` name mismatch |
| **Network** | firewall, NAT, bind address, UDP vs TCP, proxy, DNS, certificate | `connection reset`, timeouts only from one side, works on localhost only |
| **Lifecycle** | process dies with the session, stale process holding the port, log not flushed, restart loop | started but gone next call, `address already in use`, empty logs |

Everything else is usually a **protocol** problem (frame/track/sequence mismatch, publish/subscribe ordering, cache semantics) or a plain **logic** bug — reach for those only after the five above are excluded.

## Bisection ladder

Each rung isolates one variable. Stop at the first rung that fails.

```bash
ssh <host> true                                   # 1 transport+auth
ssh <host> 'echo $PATH; which <tool>'             # 2 shell and PATH
ssh <host> '<tool> --version'                     # 3 program starts at all
ssh <host> '<tool> --help'                        # 4 argument parsing
ssh <host> '<tool> <minimal args>' </dev/null     # 5 minimal real invocation
ssh <host> '<tool> <real args>'                   # 6 real invocation
```

If rung 3 works and rung 5 fails, the problem is arguments, config, input, or permissions — not the deployment. If rung 2 fails, stop trying to run the program.

## Symptom table

| Symptom | Class | First check |
| --- | --- | --- |
| `Permission denied (publickey)` | Access | `ssh -vvv`, key path, `authorized_keys`/`icacls` |
| Exit code 255 | Access | connection/auth, not your command |
| Command hangs, no output | Lifecycle / stdin | add `-n` or `</dev/null`; check for a password prompt |
| `command not found` over SSH but works in the GUI | Environment | full path, `bash -lc`, machine vs user `PATH` |
| `Address already in use` | Lifecycle | `ss -ltnp \| grep <port>`, find the stale owner |
| Process exists now, gone next call | Lifecycle | `setsid nohup` + redirected fds, or a real unit/task |
| `Exec format error` | Environment | architecture mismatch |
| `error while loading shared libraries` | Environment | `ldd <binary>`, libc family, missing runtime path |
| Firewall-ish timeout from one host only | Network | bind address (`0.0.0.0` vs `127.0.0.1`), host firewall rules |
| Listening port correct, no data | Network/Protocol | is the peer connecting? UDP blocked? wrong port/topic? |
| `Device or resource busy` | Device | another process holds the camera/serial node |
| Log is empty | Lifecycle | buffering, wrong path, redirected to the wrong file |
| Works interactively, fails automated | Environment | login shell env, TTY size, `TERM`, aliases, locale |

## Cheap discriminators

- **Localhost vs remote**: if it works on `127.0.0.1` but not from another host, it is binding or firewall, not the program.
- **Old process vs new**: capture the PID before and after. Same PID means you never restarted anything.
- **Different user**: run the same command as the SSH user, not as root, to expose permission-sensitive failures.
- **Different input**: synthetic source or tiny fixture instead of the real device; if the synthetic path works, the difference is the hardware.
- **Time**: does the failure appear at start, at a fixed interval, or under load? Each points at a different subsystem.

## Rules while debugging

- **Read the log before changing the code.** Twice.
- **One variable per attempt**, and write down what you changed.
- **Prefer rollback over incremental patching** when the last known-good state is available.
- **Never "fix" by disabling a safety check** (host-key verification, TLS validation, permission checks) to get past an error.
- If a hypothesis fails twice, it is probably wrong; return to classification instead of adding a third workaround.
- If you cannot test a layer, say so. "Could not verify" is a valid, useful conclusion — an invented cause is not.

## When to stop and ask

Escalate to the device owner rather than experimenting further when:

- The fix requires physical access (reseat cable, power cycle, hardware enable).
- The fix requires privileged, persistent, or shared-state changes (service config, firewall policy, firmware, disk).
- Credentials or certificates are missing or expired.
- The device shows signs of a hardware fault (thermal, boot failure, repeated driver-level errors).
- The change would affect other users or other tests on the same device.
