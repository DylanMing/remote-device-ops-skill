# Environment and Artifact Planning

Use this reference to turn an ad hoc remote test into a repeatable test specification. Program names are not fixed by the skill. Name each executable by the role it plays in the test, then bind that role to a concrete binary, script, package, or service for the current project.

## Test Specification Template

```yaml
test:
  name: <short-test-name>
  goal: <what must be proven>
  safety:
    forbidden_ports: [<ports-not-to-touch>]
    hardware_allowed: [camera, gpu, serial, robot-actuator]
    install_allowed: false

hosts:
  controller:
    description: <local machine or server-side coordinator>
    os: <linux/windows/macos>
  remote_device:
    ssh_alias: <host>
    os: <linux/windows/macos/embedded>
    arch: <x86_64/aarch64/arm64>
    user: <username>
    test_dir: <remote-test-directory>

dependencies:
  runtimes:
    - role: python
      required: true
      min_version: "3.10"
      probe: "python --version || python3 --version || py --version"
    - role: media-tool
      required: false
      probe: "ffmpeg -version"
  tools:
    - role: producer
      required: true
      path: <remote-path-or-package-entrypoint>
      probe: "<producer> --version || <producer> --help"
    - role: consumer
      required: false
      path: <local-or-remote-path>
      probe: "<consumer> --help"
  hardware:
    - role: camera
      required: false
      probe_windows: "ffmpeg -hide_banner -list_devices true -f dshow -i dummy"
      probe_linux: "ls -l /dev/video* || true"

artifacts:
  build_strategy: <remote-native | local-same-target | cross-compile | prebuilt | package-install>
  target: <os-arch-libc-or-language-runtime>
  package_type: <wheel | binary | archive | container | scripts>
  files:
    - local: <path-on-controller>
      remote: <path-on-device>

commands:
  probe: <one-shot environment probe>
  install: <optional install command>
  smoke: <synthetic or minimal test command>
  run: <real test command>
  stop: <stop command>
  collect_logs: <log collection command>

metrics:
  expected:
    correctness: <pass condition>
    throughput: <rate>
    latency: <budget>
    resource: <cpu/memory budget>
```

## Dependency Inspection

Always inspect before installing or copying tools:

- **OS/architecture**: `uname -a`, `uname -m`, `Get-ComputerInfo`, `PROCESSOR_ARCHITECTURE`.
- **Shell and PATH**: `echo $SHELL`, `$env:Path`, `command -v <tool>`, `where.exe <tool>`.
- **Language runtimes**: Python, Node, Java, Go, Rust, .NET, or project-specific SDKs.
- **Package managers**: apt, yum, dnf, pacman, brew, winget, choco, scoop, uv, pip, npm, cargo.
- **Drivers and hardware**: camera, GPU, serial, USB, CAN bus, robot control bus, kernel modules.
- **Network**: ports, firewall, proxy, DNS, route to relay/server, UDP/TCP availability.

Do not assume a GUI-installed dependency is visible over SSH. Verify the executable path from the SSH session that will run the test.

## Install Strategy

Use the least invasive path that satisfies the test:

1. Use existing remote dependencies if versions and paths are valid.
2. Install into a test-local directory or virtual environment when possible.
3. Use OS package managers only with user approval and record the command.
4. Use prebuilt offline installers when the remote has no network access.
5. Avoid global PATH or service changes unless the test requires them.

Typical commands:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
uv venv .venv
uv pip install -r requirements.txt
```

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
winget list Python.Python.3
where.exe python
where.exe ffmpeg
```

## Build and Packaging Strategy

Choose one:

- **Remote native build**: best when the remote has the correct compiler/toolchain and enough CPU/disk.
- **Local same-target build**: build on a machine with the same OS/architecture, then copy the artifact.
- **Cross-compile on controller/server**: use a target triple, SDK, sysroot, or cross toolchain matching the remote.
- **Language package**: build a wheel, npm package, jar, or archive and install it on the remote.
- **Single binary/script bundle**: copy only `bin/`, `config/`, `scripts/`, and data required for the test.
- **Container image**: useful for server-side roles, less useful for hardware devices unless device passthrough is solved.

Before using a copied artifact, verify:

- Target OS and architecture match.
- Dynamic library dependencies are present.
- Runtime version is compatible.
- The binary starts and prints version/help.
- The artifact can access required devices and files under the SSH user.

## Remote Test Directory Layout

Prefer a predictable, self-contained layout:

```text
<test_dir>/
  bin/          # copied binaries and project-specific tools
  scripts/      # run, stop, probe, and log collection scripts
  config/       # test configs only
  data/         # small input fixtures
  logs/         # stdout/stderr and app logs
  packages/     # wheels, installers, archives
  .venv/        # optional Python environment
```

Every long-running test should have:

- A `run_*` script that starts it.
- A `stop_*` script that finds only this test's processes.
- A stable marker string in command lines or log paths.
- A log file path that can be collected after failure.
