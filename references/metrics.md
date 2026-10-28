# Metrics Reference

Collect metrics from every role in the test: producer, service/relay, consumer, hardware device, controller, and user-facing output. Prefer windowed rates for real-time behavior and cumulative counters for totals.

## Core Metrics

- **Liveness**: process running, connection established, recent log timestamp, outputs increasing.
- **Correctness**: expected response, checksum, decoded payload, control action, state transition, or visible output.
- **Throughput**: bytes/sec, Mbps, messages/sec, frames/sec, commands/sec, samples/sec.
- **Windowed Rate**: recent 10s/30s/60s window. Do not rely only on since-start averages.
- **Drops/Loss**: missing sequence numbers, dropped frames/messages, retransmits, subscriber errors.
- **Latency**: latest, p50, p95, p99 when timestamps are available.
- **Jitter**: inter-arrival variation or p95 deviation from expected interval.
- **Resource usage**: CPU, RSS/memory, GPU/encoder load when applicable.
- **Backpressure**: queue length, buffered media duration, send/receive stalls.
- **Device health**: hardware opened, driver selected, permission status, sensor/control feedback.
- **Lifecycle**: return code, restart count, unexpected exits, stale processes after stop.

## Generic Test Indicators

Producer or remote device:

```text
required runtime found at expected path
project-specific tool starts and reports version/help
device or synthetic source opened
connection to server/controller established
messages/outputs increasing
no repeated reconnect loop
```

Service or relay:

```text
listening on expected ports
accepted remote connection
topic/session/job/stream registered
forwarded or processed units increasing
queue and memory stable enough for the test duration
```

Consumer or validator:

```text
received expected units
validated payload or side effect
recent rate near expected rate
loss and latency within budget
no decode, parse, protocol, or timeout errors
```

## Optional Media Test Indicators

Producer side:

```text
camera opened
encoder selected expected codec/resolution/fps
publisher connected
track/catalog announced
subscriber started for media track
groups or frames being served
```

Relay/subscriber side:

```text
relay listening on expected TCP/UDP ports
subscriber state running
frames increasing
dropped_frames stable
recent fps near expected fps
recent bitrate near configured bitrate
relay RSS stable enough for test duration
```

Frontend side:

```text
video visible
decoder active
FPS window value near source FPS
latest latency/jitter acceptable
no repeated reconnect or decode errors
```

## Reporting Template

```text
Remote:
  host/user:
  test directory:
  running processes:
  start command:
  stop command:

Service:
  relay URL:
  viewer URL:
  publish name/topic:
  tracks/streams:

Metrics:
  recent rate:
  throughput:
  frames/messages/commands:
  drops/loss:
  latency:
  jitter:
  CPU:
  memory:

Conclusion:
  works:
  risks:
  next fixes:
```
