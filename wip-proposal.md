Hello everyone, this is my first semconv issue proposal, so apologies in advance if I miss anything. 
Happy to restructure or move this elsewhere if a maintainer points me to a better venue.

I already have a draft PR ready locally (YAML model changes, generated markdown, and a chloggen entry), but I'd rather open this issue first to align on shape, naming, and review venue before sending it for review. Once the direction is agreed I'll open the PR and link it back here.

## Summary

I would like to propose to extend the existing stable `system.network.*` metric namespace with two sub-namespaces — `system.network.tcp.*` and `system.network.udp.*` — covering kernel-observable TCP and UDP signals (RTT, connection lifecycle, retransmits, resets, UDP errors and queue drops, etc.) that are not currently expressible in semconv. Introduce a small set of supporting attributes under `network.tcp.*`.

All new conventions would start at `development` stability, per [how-to-write-conventions](docs/how-to-write-conventions/README.md).

## Motivation

The existing `system.network.*` namespace covers transport-agnostic NIC-level counters (`system.network.io`, `system.network.connection.count`, `system.network.errors`, `system.network.packet.count`, `system.network.packet.dropped`) but does not descend into protocol-specific detail. 

There is no semconv entry for TCP RTT, TCP failed connection attempts, retransmits, resets, accept-queue overflows, or UDP error/drop categories.

These are signals that:

- can be collected from kernel sources (eBPF kprobes/tracepoints for example) on Linux, and have analogues on other OSes;
- are equally relevant to multiple kinds of agents/instrumentations (system-metrics collectors, eBPF-based agents, host-level exporters), so they fit the [scope rule](CONTRIBUTING.md#which-semantic-conventions-belong-in-this-repo) that conventions in this repo apply across multiple runtimes/components;
- are routinely re-invented per vendor today, leading to incompatible dashboards and alerts for fundamentally identical kernel quantities.

## Relationship to a parallel `network.flow.*` proposal

A separate proposal from MARIO is being drafted for network flow metrics under a `network.flow.*` root. The two efforts are intentionally separate but they will sometimes describe the same underlying phenomenon (e.g. RTT) from different points of view. I've coordinated with the author of that proposal and we've agreed:

- These are **two distinct proposals**, not one.
- Each metric name is a complete semantic convention on its own (e.g. `system.network.tcp.rtt` and `network.flow.tcp.rtt` are two separate conventions that happen to describe similar quantities from different POVs — not a shared `tcp.rtt` entry under two parent namespaces).
- Where the two proposals overlap semantically (units, attribute values, enum members), we will keep them aligned manually rather than silently diverge.

Calling this out so reviewers know the alignment is being managed out-of-band and so feedback on shared semantics can be applied consistently to both proposals. 


## Scope

In scope:

- TCP: RTT, connection success/failure, connection duration, handshake duration, retransmits, resets, accept-queue overflow, flow-control events.
- UDP: error categories, socket-queue drops, message size distribution. Datagram *counts* are intentionally **not** added — they reuse the existing `system.network.packet.count` with `network.transport=udp`.
- New attributes describing the above (handshake role, retransmit type, reset cause, flow-control event).

Out of scope (for this proposal):

- L7 protocol metrics (HTTP/gRPC/etc. — already covered by their own conventions).
- Per-flow / per-socket cardinality. Metrics are aggregate.
- Anything that requires a stable userspace-only signal (e.g. application-layer errors).

## Namespace layout

```
system.network.*                              — existing stable root
  system.network.io                           — existing
  system.network.packet.count                 — existing
  system.network.packet.dropped               — existing
  system.network.errors                       — existing
  system.network.connection.count             — existing

  system.network.tcp.*                        — TCP metrics (NEW)
    system.network.tcp.rtt
    system.network.tcp.connection.successes
    system.network.tcp.connection.failures
    system.network.tcp.connection.duration
    system.network.tcp.handshake.duration
    system.network.tcp.retransmits
    system.network.tcp.resets
    system.network.tcp.accept_queue.overflows
    system.network.tcp.flow_control.events

  system.network.udp.*                        — UDP metrics (NEW)
    system.network.udp.errors
    system.network.udp.queue.drops
    system.network.udp.message.size

network.*                                     — attribute root (unchanged)
  network.transport                           — existing
  network.io.direction                        — existing
  network.connection.state                    — existing
  network.tcp.handshake.role                  — NEW (client | server)
  network.tcp.retransmit.type                 — NEW (fast | tail_loss_probe | timeout | spurious)
  network.tcp.reset.cause                     — NEW, opt-in (application | timeout | unreachable | refused)
  network.tcp.flow_control.event              — NEW (zero_window | window_full)
```

## Naming principles

Following [naming.md](docs/general/naming.md) and existing semconv precedent:

1. **Nest under `system.network.*`** (sub-namespacing inside an existing stable root, mirroring `jvm.gc.*` / `jvm.memory.*`).
2. **No `.total` suffix** on counters (Prometheus exporter adds it).
3. **Units via UCUM**, not in the name: `By`, `s`, and grammatically-singular `{segment}`, `{connection}`, `{event}`, `{error}`, `{datagram}` for counts.
4. **Attributes over separate metrics** when aggregation across the attribute values is still meaningful; separate metrics when the events are semantically distinct.
5. **Reuse stable attributes** (`error.type`, `network.io.direction`, `network.transport`, `network.connection.state`) before introducing new ones.

Note: metric names sit under `system.network.*` while attribute names sit under `network.*`, matching the existing layout (e.g. `system.network.connection.count` carries `network.connection.state`).

## Proposed metrics — TCP

### `system.network.tcp.rtt`

- **Instrument**: Histogram
- **Unit**: `s`
- **Brief**: Each observation represents the smoothed RTT of a single TCP connection, recorded at the time of close.
- **Source examples**: `tcp_sock.srtt_us` read at `tcp_close` (Linux); equivalent stack-reported RTT on other OSes.
- **Attributes**: none required.

### `system.network.tcp.connection.successes`

- **Instrument**: Counter
- **Unit**: `{connection}`
- **Brief**: TCP connections that completed handshake and reached `ESTABLISHED`.
- **Attributes**: `network.tcp.handshake.role` (`client`|`server`).

### `system.network.tcp.connection.failures`

- **Instrument**: Counter
- **Unit**: `{connection}`
- **Brief**: TCP connection establishment attempts that did not reach `ESTABLISHED`.
- **Attributes**:
  - `error.type` (reused stable attribute) — proposed value set: `refused | reset | timed_out | host_unreachable | net_unreachable | _OTHER`.
  - `network.tcp.handshake.role` (`client`|`server`).

### `system.network.tcp.connection.duration`

- **Instrument**: Histogram
- **Unit**: `s`
- **Brief**: Time from connection establishment to close, recorded at close.
- **Attributes**: `network.tcp.handshake.role`.
- **Note**: Spans from `ESTABLISHED` to close. Reflects end-to-end session length, not just connection setup.

### `system.network.tcp.handshake.duration`

- **Instrument**: Histogram
- **Unit**: `s`
- **Brief**: Time from `SYN` sent (active open) or `SYN` received (passive open) to handshake completion (`ESTABLISHED`).
- **Attributes**: `network.tcp.handshake.role`.
- **Note**: Useful as a server-side health signal: prolonged handshake durations can indicate listener saturation or backpressure on the `SYN` queue.

### `system.network.tcp.retransmits`

- **Instrument**: Counter
- **Unit**: `{segment}`
- **Brief**: TCP segments retransmitted.
- **Attributes**: `network.tcp.retransmit.type` (`fast | tail_loss_probe | timeout | spurious`), aligned with kernel counters `TCPFastRetrans`, `TCPLossProbes`, `TCPTimeouts`, `TCPSpuriousRetransmits`.
- **Note**: `network.io.direction` is intentionally omitted — retransmits are a sender-side phenomenon.

### `system.network.tcp.resets`

- **Instrument**: Counter
- **Unit**: `{segment}`
- **Brief**: RST segments observed.
- **Attributes**:
  - `network.io.direction` (`receive`|`transmit`).
  - `network.tcp.reset.cause` (optional; `application | timeout | unreachable | refused`) when cheaply inferable.
- **Note**: Only counts RST segments observed *outside* connection establishment. RSTs that cause a connection attempt to fail are counted under `system.network.tcp.connection.failures` and MUST NOT also be counted here, to avoid double-counting the same on-the-wire event.

### `system.network.tcp.accept_queue.overflows`

- **Instrument**: Counter
- **Unit**: `{event}`
- **Brief**: Listen-socket accept-queue overflow events (`sk_ack_backlog > sk_max_ack_backlog` on Linux). Named `accept_queue` rather than the Linux-colloquial "listen overflow" to keep the convention portable.
- **Attributes**: none required.

### `system.network.tcp.flow_control.events`

- **Instrument**: Counter
- **Unit**: `{event}`
- **Brief**: TCP flow-control notifications. Folded into a single metric since Zero-Window (an explicit stop signal from the receiver) and Window-Full (the sender reaching the receiver's limit) both indicate the transmission has stalled due to buffer exhaustion.
- **Attributes**: `network.tcp.flow_control.event` (`zero_window`|`window_full`), `network.io.direction`.
- **Note**: The two event values map to distinct operator workflows: `zero_window` is observed when investigating receiver-side memory pressure or slow consumers; `window_full` is observed when investigating sender-side throughput stalls. That's why the distinction is preserved as an attribute rather than discarded.

## Proposed metrics — UDP

UDP is stateless: no connections, no retransmits, no handshake. Signals of interest are error categories and payload sizing.

### `system.network.udp.errors`

- **Instrument**: Counter
- **Unit**: `{error}`
- **Brief**: UDP-level error events.
- **Attributes**:
  - `error.type` (reused) — proposed values: `checksum | port_unreachable | send_buffer | receive_buffer`. `port_unreachable` is chosen to match ICMP terminology rather than the Linux-specific `NoPort`.
  - `network.io.direction` where applicable.

### `system.network.udp.queue.drops`

- **Instrument**: Counter
- **Unit**: `{datagram}`
- **Brief**: Datagrams dropped due to socket queue exhaustion. Distinct from interface-level drops covered by `system.network.packet.dropped`.
- **Attributes**: `network.io.direction`.

### `system.network.udp.message.size`

- **Instrument**: Histogram
- **Unit**: `By`
- **Brief**: Size distribution of UDP message payloads.
- **Attributes**: `network.io.direction`.

## Proposed new attributes

| Attribute | Type | Values | Applies to |
|---|---|---|---|
| `network.tcp.handshake.role` | enum | `client`, `server` | TCP connection/handshake metrics |
| `network.tcp.retransmit.type` | enum | `fast`, `tail_loss_probe`, `timeout`, `spurious` | `system.network.tcp.retransmits` |
| `network.tcp.reset.cause` | enum (optional) | `application`, `timeout`, `unreachable`, `refused` | `system.network.tcp.resets` |
| `network.tcp.flow_control.event` | enum | `zero_window`, `window_full` | `system.network.tcp.flow_control.events` |

All four would be defined under `model/network/registry.yaml` with
`development` stability.

## Open questions for reviewers

1. Is `system.network.tcp.*` / `system.network.udp.*` the right home, or would maintainers prefer a sibling `system.tcp.*` / `system.udp.*` namespace? (Argument for nesting: keeps all kernel-network signals under one root and matches `jvm.gc.*` precedent. Argument against: TCP/UDP are transport, not strictly "network" in the layer-3 sense.)

2. For TCP failure reasons, is reusing `error.type` with this value set acceptable, or should there be a TCP-specific `network.tcp.connection.error_code`?

3. `system.network.tcp.flow_control.events`: keep folded, or split into two metrics (`zero_window`, `window_full`)?

4. **Successes + failures: one counter or two?** Currently I have two counters — `system.network.tcp.connection.successes` and `system.network.tcp.connection.failures`. Would maintainers prefer collapsing them into a single counter, modelled like `db.client.operation.duration`? Sketch:

    ```
    system.network.tcp.connections    Counter   {connection}
      attributes:
        - error.type                  (unset on success; populated on failure)
        - network.tcp.handshake.role
    ```

    Pros: idiomatic semconv, single PromQL expression for success rate. Cons: failures-only attribute set is mixed with success path. Happy to go either way.

5. **`network.tcp.retransmit.type` portability across OSes.** The four values (`fast`, `tail_loss_probe`, `timeout`, `spurious`) all map to Linux kernel counters. Windows only exposes the equivalent of `fast` and `timeout`; macOS/BSD support is unclear. If a user runs `sum by (type) (rate(system.network.tcp.retransmits[5m]))` on a mixed Linux/Windows fleet, the breakdowns won't be comparable. Three options:

    - Keep all four values and document the OS limitation in the attribute's `note:`.
    - Mark `spurious` and `tail_loss_probe` as opt-in / best-effort.
    - Split: keep `fast` / `timeout` portable, move `spurious` / `tail_loss_probe` into a Linux-specific attribute.

    My lean is option 1; would like maintainer guidance.

6. **Possible double-counting between `system.network.tcp.resets` and `system.network.tcp.connection.failures`.** A connection refused on the wire produces both a RST segment and a failed connection attempt. With the current proposal, a single on-the-wire event could increment `system.network.tcp.connection.failures` (with `error.type=refused`) **and** `system.network.tcp.resets` (with `network.tcp.reset.cause=refused`). I've added a `note:` to `tcp.resets` saying it MUST NOT count RSTs that already incremented `connection.failures`. Is that the right boundary, or should the two metrics be allowed to overlap and let consumers deduplicate?

7. **`system.network.tcp.accept_queue.overflows` portability.** The concept is portable (Linux, FreeBSD, Windows all have *some* listener-overflow signal), but the source counters and exact semantics differ. Two options:

    - Keep it under `system.network.tcp.*` with a `note:` saying *"On Linux, sourced from `LINUX_MIB_LISTENOVERFLOWS` / `LINUX_MIB_LISTENDROPS`. On other operating systems, instrumentations MAY emit this metric using equivalent counters where available."* — current proposal.
    - Move it to `system.network.linux.tcp.accept_queue.overflows`, mirroring the `system.memory.linux.*` precedent.

8. **`system.network.udp.message.size` histogram bucketing.** UDP payload sizes are heavily multimodal — clusters around small control-plane sizes (DNS ~50–512 B), MTU-sized payloads (~1400 B), occasional jumbo frames (~9000 B), and fragmented payloads. Default exponential buckets won't distinguish these well. Should the metric carry a `note:` advising operators about multimodal distributions, or is bucket selection out of scope for semconv?

9. **`network.interface.name` as an attribute on selected metrics.** Currently only the host is associated as an entity. Operators commonly want to break RTT, retransmits, resets, and message-size distributions out per interface (`eth0` vs `wg0` vs `lo` produce very different distributions). Proposal: add `- ref: network.interface.name` to:

    - `system.network.tcp.rtt`
    - `system.network.tcp.retransmits`
    - `system.network.tcp.resets`
    - `system.network.udp.message.size`

    Skip it on connection-lifecycle, accept-queue, flow-control, and UDP error/drop metrics where the interface dimension doesn't really vary. Confirm or push back?

10. Which SIG should I bring this proposal to? The PR will touch `/model/network/registry.yaml` and `/model/system/`, so CODEOWNERS will auto-request the system, HTTP, and security approver teams — but I'd like to know if there's a specific SIG meeting (Semantic Conventions, System, or otherwise) where I should also present it. Happy to attend whichever is most useful.


## Cross-implementation prototyping

Per [how-to-write-conventions § Prototyping](docs/how-to-write-conventions/README.md#prototyping),
two of the proposed metrics are already implemented in
[`open-telemetry/opentelemetry-ebpf-instrumentation`](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation):

- `system.network.tcp.rtt` (currently emitted as `obi.stat.tcp.rtt`,
  to be renamed once this proposal lands).
- `system.network.tcp.connection.failures` (currently emitted as
  `obi.stat.tcp.failed.connections`).

The remaining metrics are in progress in the same repo.


## PR plan (once agreement is reached)

The actual PR would touch:

- `model/network/registry.yaml` — add the four new attributes.
- `model/system/metrics.yaml` — add the new TCP/UDP metric groups
  (or split into `model/system/network-metrics.yaml` if maintainers prefer).
- `docs/system/system-metrics.md` — add new sections under the existing
  `system.network` block, with `<!-- semconv ... -->` markers for generation.
- `.chloggen/<branch>.yaml` — single entry, `component: system`,
  describing the new metrics and attributes.
- Run `make generate-all`, `make check-policies`, `make check` before pushing.
