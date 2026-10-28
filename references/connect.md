# Connect and Authenticate

The whole workflow rests on one property: the agent can run commands without a human at the keyboard. Establish that property deliberately, then freeze it into configuration.

## Three-stage verification

Run them in order and stop at the first failure — each stage has a different owner.

```bash
# 1. Transport: is the port reachable at all?
nc -vz <host> 22                 # PowerShell: Test-NetConnection <host> -Port 22

# 2. Authentication: does key auth succeed without prompts?
ssh -o BatchMode=yes -o ConnectTimeout=10 <host> 'echo ok'

# 3. Execution context: what environment will real commands see?
ssh -o BatchMode=yes <host> 'whoami; hostname; pwd; echo "$SHELL"; echo "$PATH"'
```

`BatchMode=yes` is the test: it makes SSH fail instead of prompting. Anything that only works interactively is not usable by an agent.

## Inspect the effective configuration

`~/.ssh/config` plus defaults decide what actually connects; do not guess.

```bash
ssh -G <host> | sed -n '1,120p'      # resolved HostName, Port, User, IdentityFile, ProxyJump, Algorithms
ssh -vvv -o BatchMode=yes <host> true 2>&1 | grep -Ei 'identity|offering|authenticat|server host key|kex'
```

Record the resolved `HostName`/`Port`/`User`/`IdentityFile` in the report — "it works on my machine" almost always means a different alias resolved differently.

## Key-based login

Generate once on the controller (ed25519, no passphrase for automation, or load the key into an agent):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N '' -C 'agent@controller'
```

### POSIX target

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <host>
# or manually:
ssh <host> 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys' < ~/.ssh/id_ed25519.pub
```

Then prove it with no fallback: `ssh -o PreferredAuthentications=publickey -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 <host> echo ok`.

Permissions are the usual culprit — `~` must not be group/world writable, `~/.ssh` must be `700`, `authorized_keys` `600`. Check `sshd` logs (`/var/log/auth.log`, `journalctl -u ssh`) rather than guessing.

### Windows target

Windows OpenSSH reads `authorized_keys` from two possible places; the account's group decides which.

- Regular user: `C:\Users\<User>\.ssh\authorized_keys`
- Member of `BUILTIN\Administrators` (when `Match Group administrators` is active): `C:\ProgramData\ssh\administrators_authorized_keys`

ACL restrictions matter as much as the file path. Full procedure, `icacls` commands, and encoding pitfalls: `references/windows.md`.

## Host keys

First contact asks for confirmation; an agent cannot answer it. Either pre-trust the key out of band, or accept it once explicitly:

```bash
ssh-keyscan -t ed25519 <host> >> ~/.ssh/known_hosts   # verify the fingerprint separately if it matters
```

`Host key verification failed` after a device reimage is expected: remove the stale entry with `ssh-keygen -R <host>` after confirming the change is real. Never disable `StrictHostKeyChecking` globally to silence it.

## Jump hosts and bastions

```bash
ssh -J <user>@<bastion> <user>@<target>            # one hop
ssh -o ProxyJump=<bastion> <target>                # or pin it in config
```

`ProxyJump` needs no `nc` on the bastion and is the preferred form. For chains, list them comma-separated. Test the bastion and the target separately so a failure points at the right hop.

## Keep the channel healthy

Long runs die of idle timeouts, not of application bugs.

```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o TCPKeepAlive=yes <host> '<long command>'
```

For many short invocations, multiplex the connection — one handshake, then near-instant commands, and a shared control socket you can shut down:

```
# ~/.ssh/config
Host <alias>
    HostName <addr>
    User <user>
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

`ControlMaster` is POSIX-side (and Windows via Git Bash); native Windows OpenSSH lacks it. Close a stuck master with `ssh -O exit <alias>`.

## Error → meaning

| Symptom | Likely cause | First check |
| --- | --- | --- |
| `Connection refused` | `sshd` not running, wrong port, firewall reject | `Test-NetConnection`/`nc`, service status on target |
| `Connection timed out` | route, firewall drop, wrong address, VPN down | ping/TCP, corporate firewall, bastion path |
| `Permission denied (publickey)` | key not offered, not in `authorized_keys`, bad perms/ACL | `ssh -vvv`, `authorized_keys` contents, `icacls`/`chmod` |
| `Permission denied (publickey,password,keyboard-interactive)` | key auth failed and it fell back to password | fix key auth; the agent cannot type a password |
| `Host key verification failed` | device reimaged, key rotated, MITM | compare fingerprints, then `ssh-keygen -R` |
| `kex_exchange_identification` | port is not SSH, or `MaxStartups`/fail2ban throttling | what is listening on that port, server logs |
| `Too many authentication failures` | every key offered before the right one | `IdentitiesOnly yes` + explicit `-i` |
| `Pseudo-terminal will not be allocated` | no TTY (expected for agents) | use a script file, not `-t` with interactive tools |
| Hangs after connect | a program is waiting on stdin | add `-n` or `</dev/null` |
| Drops mid-run | idle timeout / NAT | `ServerAliveInterval`, or detach with `setsid nohup` |
| `Bad owner or permissions on ~/.ssh/config` | config perms too open | `chmod 600 ~/.ssh/config` |

## Credentials

Use keys. If a password is truly the only option, it must come from a secret store and be fed non-interactively — and even then, prefer replacing it with a key. Never paste a password into a command line, a script that gets committed, or the report.

Note the passphrase problem explicitly: an encrypted key needs an agent (`ssh-add`) or a non-interactive resolution path. If neither exists, the key is not usable by the agent, regardless of what works in a human terminal.
