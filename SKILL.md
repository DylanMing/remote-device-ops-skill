---
name: remote-device-test
description: Generic remote device test workflow over SSH. Use when Codex needs to connect to remote Linux, Windows, macOS, or embedded targets; inspect or install dependencies; build, cross-compile, package, and copy test artifacts; run remote services or hardware tests; diagnose SSH/auth/quoting/process issues; collect logs; and report correctness, latency, throughput, loss, CPU, memory, device, or media metrics.
---

# Remote Device Test

## Core Workflow

Use a staged workflow. Do not start the real test until access, environment, artifacts, and safety boundaries are clear.

1. Confirm target and constraints:
   - Host alias or address, user, OS, and whether SSH is already configured.
   - Ports and services that must not be touched.
   - Whether the remote test can use hardware such as camera, microphone, GPU, or robot actuators.
   - Where temporary files and logs should live on the remote device.
   - Whether Codex may install dependencies, change services, open firewall rules, or only inspect.

2. Define the test specification:
   - Treat project-specific programs as roles, not fixed names: `<producer>`, `<consumer>`, `<relay>`, `<device-tool>`, `<robot-control>`, `<analyzer>`, etc.
   - List required runtimes, package managers, executable roles, configs, data files, ports, credentials, hardware, and expected metrics.
   - Choose where each role runs: local controller, remote device, relay/server, or another host.
   - Read `references/environment-and-artifacts.md` for a reusable manifest template.

3. Validate SSH non-interactively:
   - Run a no-op command first, for example `ssh <host> echo ok`.
   - If Codex will automate the remote host, require public-key login or another non-interactive method.
   - If SSH asks for a password in Codex, stop and explain that the command has no interactive TTY.
   - For Windows OpenSSH details, read `references/windows-ssh.md`.

4. Inspect the remote environment:
   - OS version, user identity, current directory, shell type.
   - Required runtimes named by the test: Python/uv, Node/npm, Java, Rust/Cargo, ffmpeg, Docker, vendor SDKs, device drivers, or other tools.
   - Required executable roles and their exact paths. Do not assume PATH is the same in SSH, GUI, and scheduled tasks.
   - Hardware devices and permissions.
   - Existing relevant processes and ports.

5. Decide build, install, and packaging strategy:
   - Use existing remote tools when present and versions are acceptable.
   - If missing, install on the remote only when allowed and record the package manager or installer used.
   - If the remote lacks a toolchain, build locally for the same OS/architecture or cross-compile on the server, then transfer the artifact.
   - Package the smallest reproducible unit: wheel, archive, single binary, container image, script bundle, or installer.
   - Verify every copied executable with a version/help/smoke command on the remote host.

6. Transfer only necessary artifacts:
   - Prefer a dedicated remote test directory.
   - Use `scp` or `rsync` for scripts and small files.
   - Avoid copying secrets, caches, virtualenv internals, nested `.git`, or large build artifacts unless needed.
   - Record exact remote paths in the final summary.

7. Run a low-risk smoke test before the real test:
   - Use a synthetic or loopback source when possible before real hardware.
   - Verify the remote command can start, connect, exchange a minimal message, or call the target API.
   - Only then use real hardware or robot control.

8. Run the real test with observable logs:
   - Prefer a foreground command while debugging.
   - For long-running tests, use a service manager appropriate to the OS only after the foreground command is proven.
   - Always capture stdout/stderr to files or a terminal stream.
   - Keep process IDs and stop commands handy.

9. Validate from both sides:
   - Remote side: process list, logs, hardware handle, role-specific logs, return codes.
   - Local/controller side: receiver status, messages or outputs received, correctness, loss, latency, throughput, memory, and CPU.
   - If the test involves media or UI, verify actual playback or visible output and not only transport counters.

10. Summarize reproducibly:
   - Commands run, files copied, remote paths, service URLs, process IDs, and stop commands.
   - What succeeded, what is still fragile, and which metrics were measured.

## Remote Command Rules

- Use one-shot commands for environment checks.
- Use script files for complex Windows PowerShell or shell quoting.
- Avoid relying on interactive password prompts, GUI prompts, or manual input from Codex.
- Use full paths for project-specific executables unless PATH was verified inside the SSH session.
- Treat SSH session lifetime as part of the test design. If a process dies when SSH exits, say so and switch to a real service manager or ask the user to keep a foreground terminal open.
- Do not leave hidden remote processes running without telling the user how to stop them.

## Diagnostics

When a remote test fails, classify the failure before changing code:

- **Access**: SSH route, username, host key, public key, password prompt, TTY.
- **Environment**: missing binary, wrong PATH, wrong virtualenv, missing dependency, architecture mismatch.
- **Device**: camera/device name, permissions, busy device, driver, unsupported format.
- **Network**: firewall, UDP blocked, relay URL, certificate endpoint, NAT, wrong bind address.
- **Protocol**: track/name mismatch, codec/container mismatch, subscriber starts too early/late, cache semantics.
- **Lifecycle**: background process dies with SSH, scheduled task cannot see user environment, logs redirected wrong.

Use `references/checklists.md` for concrete command templates and `references/metrics.md` for metrics to collect.

## Reporting

Keep the final report compact but complete:

- Remote device and user.
- Test directory and artifacts copied.
- Exact start command and stop command.
- Key logs proving the connection.
- Metrics table or short JSON snippets.
- Risks and next fixes, ordered by production impact.
