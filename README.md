# Remote Device Ops

A portable agent skill for operating remote devices over SSH: connect, profile, run, deploy, verify, report.

It is **documentation, not code**. An agent loads `SKILL.md` when a task matches its `description`, then follows the workflow and pulls in a `references/` file only when that specific problem comes up.

## Layout

```text
SKILL.md                  entry point: invariants, workflow, routing table, hard rules
agents/                   (removed) Codex-only UI metadata — recoverable from git history
references/
  connect.md              transport, keys, host keys, jump hosts, keepalive, error table
  environment.md          one-shot device profiling: OS, arch, PATH, ports, hardware
  execute.md              quoting layers, stdin, exit codes, timeouts, sudo, detached processes
  transfer.md             scp / rsync / tar, exclusions, integrity, arch compatibility
  verify.md               evidence ladder, metric classes, false positives, report template
  diagnose.md             five failure classes, bisection ladder, symptom table
  windows.md              Windows OpenSSH: keys, ACLs, PATH, quoting, scheduler, services
```

## Design

- **Transport-agnostic content, SSH-only channel.** Only SSH is supported today; `connect.md` is the single place to add another transport.
- **Roles, not product names.** Programs are referred to as `<target>`, `<service>`, `<tool>`, `<producer>`, `<consumer>`; bind them per task.
- **Progressive disclosure.** `SKILL.md` stays small enough to load on every trigger; depth lives in `references/`.
- **Every claim has an evidence rung.** See `verify.md`: started, healthy, progressing, correct, on budget, contained.

## Install

The front matter uses only the common fields (`name`, `description`), so the same directory works as a Claude Code / Claude Skills skill, a Codex skill, or any loader that understands `SKILL.md`.

```powershell
# Claude Code / user-level skills on Windows
Copy-Item -Recurse . "$env:USERPROFILE\.claude\skills\remote-device"

# Claude Code on macOS / Linux
cp -r . ~/.claude/skills/remote-device
```

Most loaders expect the directory name to match `name:` in the front matter. This directory is currently named `remote-device-test`; if you rename it to `remote-device` (recommended, since the scope is operations rather than testing), update `name:` in `SKILL.md` in the same commit:

```powershell
git mv . ..\remote-device      # or rename in Explorer, then edit SKILL.md name:
```

## Extending

- Adding a transport (WinRM, ADB, serial): new `references/<transport>.md` plus a row in the routing table. Do not fork `SKILL.md`.
- Adding a workflow (a test campaign, an inventory sweep): add a short worked shape under "Worked shapes" in `SKILL.md`, and detail it in a reference only if it needs more than ~15 lines.
- Keep commands as templates. Placeholders (`<host>`, `<workspace>`, `<marker>`) are intentional; concrete values belong in the task, not the skill.

## History

This started as a media/streaming test skill (MoQ publisher/relay/subscriber on Windows targets) and was generalized into device operations. The earlier rule set is still in git history (`git log --all -- references/`).
