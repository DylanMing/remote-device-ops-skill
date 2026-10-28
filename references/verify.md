# Verify, Measure, Report

"Done" is a claim about the world, not about your command history. Earn it with evidence, then write down what the evidence does and does not cover.

## The evidence ladder

Climb in order; state which rung you actually reached.

| Rung | Question | Typical proof |
| --- | --- | --- |
| 1. Started | Did the process/unit/connection come up? | exit code, `ps`, `systemctl status`, `Get-Process`, listening socket |
| 2. Healthy | Is it still up, and not restarting in a loop? | uptime, restart count, no repeated reconnect lines in the log |
| 3. Progressing | Is the payload actually moving? | byte/message/frame counters increasing, rows inserted, files created |
| 4. Correct | Is the content what was promised? | checksum, decoded payload, schema validation, rendered output seen |
| 5. On budget | Does it meet the stated numbers? | windowed rate, p50/p95/p99 latency, loss, CPU/RSS |
| 6. Contained | Did anything leak? | clean stop, no orphan PIDs, ports released, temp files removed |

A run that stops at rung 2 is "started, unverified". Say so explicitly instead of implying success.

## Prefer windowed rates over lifetime averages

A counter divided by uptime hides every stall. Stalls are the failure mode that matters.

```bash
# sample a counter twice, ten seconds apart, and report the delta
ssh <host> 'a=$(grep -F "<counter>" <log> | tail -1); sleep 10; b=$(grep -F "<counter>" <log> | tail -1); echo "$a"; echo "$b"'
```

Rules:

- Report **recent-window** throughput (10 s / 30 s / 60 s) plus the cumulative total, never the average alone.
- Watch for counters that plateau, and for counters that only reset on restart (a reset is a restart — count those).
- Compare the recent rate against the expected rate; "some traffic" is not a pass.

## Metric classes worth collecting

- **Liveness** — PID alive, unit active, socket listening, last log timestamp advancing.
- **Correctness** — checksum match, expected response, valid frame/message/row, control action reached the device, output visible.
- **Throughput** — bytes/s, Mbps, messages/s, frames/s, commands/s.
- **Latency** — latest, p50, p95, p99 when timestamps exist on both ends; note clock skew if the ends differ.
- **Jitter / stability** — inter-arrival variation, p95 deviation from the nominal interval.
- **Loss** — missing sequence numbers, dropped frames/messages, retransmits, subscriber errors.
- **Resource** — CPU %, RSS, GPU/encoder load, disk growth, handle/thread counts.
- **Backpressure** — queue depth, buffered duration, send stalls, blocked writers.
- **Device health** — device node opened, driver/bus selected, permission state, sensor or actuator feedback.
- **Lifecycle** — exit code, restart count, unexpected exits, orphans left after stop.

Collect from **every** role: producer, service/relay, consumer, controller, and the human-visible result. A green producer with a starving consumer is a broken pipeline.

## False positives to rule out

These look like success and are not:

- Exit 0 from a launcher or wrapper whose actual child died immediately.
- A log line from the **previous** run — check the timestamp is after your start time, or truncate the log first.
- `systemctl is-active` on a unit stuck in `activating` or crash-looping.
- A listening port owned by the **old** process that never exited.
- Counters that only ever accumulate (so "increasing" is trivially true).
- A pipeline reporting frames while the viewer shows black — verify the visible/decoded result, not just transport counters.
- `ps` evidence from a previous session's marker string matching the same pattern.
- Buffer flush lag: a program that logs progress but has not `fflush`/committed yet.

## Comparing runs

For anything you run more than once, fix these so results are comparable: same build hash, same config file, same source (synthetic vs real), same duration, same measurement window, same machine state (nothing else on the port). Note the deltas when they are not identical — otherwise "it got faster" is a rumor.

## Report template

```text
Task:       <what was asked>
Device:     <host> / <user> / <os-arch>       session: <ssh invocation>
Workspace:  <remote paths touched, created, deleted>
Artifacts:  <files copied in/out, with remote destinations>
Started:    <command>  -> pid/unit <id>  -> logs <path>
Stopped:    <stop command, or "LEFT RUNNING — stop with: ...">

Evidence:
  started:      <exit code, process/unit state, port>
  progressing:  <counters over window>
  correct:      <checksum / payload / visible output>
  budget:       <rate, latency p50/p95/p99, loss, CPU, RSS>

Conclusion: works | works with caveats | failed — <one sentence>
Risks:      <fragile parts, unverified claims, environment drift>
Next:       <the single most valuable follow-up>
```

Keep it short enough that someone can act on it. Numbers without units, and units without windows, are decoration.
