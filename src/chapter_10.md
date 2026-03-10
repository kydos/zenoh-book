# Quality of Service

This chapter covers Zenoh's QoS knobs: reliability, priority, congestion control, and express mode. These are set per-publisher or per-put.

## Overview

Zenoh QoS controls four independent axes:

1. **Reliability** — should delivery be acknowledged?
2. **Priority** — what traffic class?
3. **Congestion control** — what happens when buffers are full?
4. **Express** — should batching be bypassed?

These can be set on `declare_publisher()`. Reliability, priority, and congestion control can also be overridden per-put on a declared publisher. Express is a publisher-level setting only.

The defaults are designed for high-throughput telemetry: best-effort, normal priority, drop on congestion, with batching enabled. Override them when your data has different requirements.

---

## Reliability

### What it is

Reliability controls whether Zenoh retransmits messages that are lost in transit.

- `Reliability::BestEffort` (default) — no retransmission. The message is sent once; if a packet is dropped, the receiver never learns of it.
- `Reliability::Reliable` — acknowledged delivery. Lost messages are retransmitted until acknowledged or the session is torn down.

### When to use it

Use `BestEffort` for high-frequency telemetry where a missed sample is acceptable and low overhead matters: sensor streams, video feeds, log tails.

Use `Reliable` for commands, configuration changes, software deployments, and any data where loss causes incorrect behavior.

### How to use it

```rust
use zenoh::{qos::Reliability, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("control/cmd")
        .reliability(Reliability::Reliable)
        .await
        .unwrap();
    publisher.put("engage").await.unwrap();
}
```

### What to watch out for

Reliability is negotiated end-to-end. If any hop on the path uses a transport that does not support reliability (e.g., UDP with `BestEffort`), the guarantee degrades to `BestEffort` on that segment. Reliable delivery over a UDP-only path is not possible; use TCP or TLS transports for reliable sessions. Verify your transport configuration when `Reliable` is required.

---

## Priority

### What it is

Priority assigns a traffic class to messages. Seven levels exist, in decreasing order of priority:

1. `Priority::RealTime`
2. `Priority::InteractiveHigh`
3. `Priority::InteractiveLow`
4. `Priority::DataHigh`
5. `Priority::Data` (default)
6. `Priority::DataLow`
7. `Priority::Background`

Higher-priority messages are scheduled before lower-priority ones in the router queues. Priority is respected end-to-end: every Zenoh router on the path schedules by priority class.

### When to use it

Use priority in mixed-criticality systems where control messages must not be delayed by bulk data transfers. For example, a robot controller that publishes both safety-critical brake commands and low-priority telemetry benefits from assigning `RealTime` to the brake publisher and `Background` to the telemetry publisher.

### How to use it

```rust
use zenoh::{qos::Priority, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let realtime = session
        .declare_publisher("actuator/brake")
        .priority(Priority::RealTime)
        .await
        .unwrap();

    let telemetry = session
        .declare_publisher("telemetry/gps")
        .priority(Priority::DataLow)
        .await
        .unwrap();

    realtime.put("brake_now").await.unwrap();
    telemetry.put("48.8566, 2.3522").await.unwrap();
}
```

### What to watch out for

Priority scheduling only takes effect when there is contention. On an idle network, all priorities are equivalent. Priority does not guarantee a latency bound; it only influences ordering relative to other messages in the queue. Do not use priority as a substitute for network capacity planning.

---

## Congestion Control

### What it is

Congestion control determines what Zenoh does when the output buffer is full and a new message arrives.

- `CongestionControl::Drop` (default) — the new message is discarded. The sender is not blocked.
- `CongestionControl::Block` — the send call blocks until buffer space becomes available.

### When to use it

Use `Drop` for data that can be replaced by the next sample: video frames, sensor readings, position updates. A dropped frame is acceptable if the next one arrives promptly.

Use `Block` for data where every message must be delivered: financial transactions, command sequences, firmware chunks. Dropping any message is a correctness error.

### How to use it

```rust
use zenoh::{qos::CongestionControl, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("stream/video")
        .congestion_control(CongestionControl::Drop)
        .await
        .unwrap();
    publisher.put(vec![0u8; 1024 * 100]).await.unwrap();
}
```

### What to watch out for

`CongestionControl::Block` can cause the publisher to stall indefinitely if the consumer is stopped or the network link is saturated. Only use `Block` with `Reliability::Reliable` and in situations where you can tolerate unbounded producer latency. In async code, a blocked send occupies the executor task until the buffer drains. If all executor threads are stalled on blocked sends, the application deadlocks. Consider using a timeout around blocked publishers in production code.

---

## Express Mode

### What it is

By default, Zenoh batches small messages to amortize per-packet overhead and improve throughput. Express mode disables this batching and sends each message immediately after it is queued.

### When to use it

Use `express(true)` when latency matters more than throughput: IMU data at 1 kHz, control signals in a feedback loop, RTT-sensitive request-response protocols. The throughput penalty relative to batched mode is roughly 10-30% for small messages, depending on message size and network conditions.

### How to use it

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("sensor/imu")
        .express(true)
        .await
        .unwrap();
    publisher.put(vec![0u8; 48]).await.unwrap();
}
```

### What to watch out for

Express mode trades throughput for latency. Enabling it on a high-volume stream that is not latency-sensitive increases CPU and network overhead without benefit. Measure the actual latency improvement in your environment before enabling it unconditionally. On loopback or gigabit LAN, the difference may be negligible.

---

## Setting QoS on Individual Puts

### What it is

A declared publisher carries default QoS settings. These defaults can be overridden on a per-put basis by chaining QoS methods on the `put` builder. The override applies only to that one publication; subsequent puts revert to the publisher defaults.

### When to use it

Use per-put overrides when a single publisher emits messages of varying criticality. For example, a publisher that normally sends background telemetry might occasionally emit a high-priority alert.

### How to use it

```rust
use zenoh::{qos::Priority, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("demo/data")
        .await
        .unwrap();

    // High-priority put
    publisher
        .put("critical")
        .priority(Priority::RealTime)
        .await
        .unwrap();

    // Background put using publisher defaults
    publisher
        .put("background data")
        .priority(Priority::Background)
        .await
        .unwrap();
}
```

### What to watch out for

Per-put QoS does not override the publisher's reliability setting; that is fixed at declaration time. Only priority, congestion control, and express can be set per-put. If you need to send the same key expression with different reliability levels, declare two separate publishers.

---

## QoS Summary Table

| Setting | Values | Default | Use case |
|---|---|---|---|
| Reliability | `BestEffort`, `Reliable` | `BestEffort` | Telemetry vs. commands |
| Priority | `RealTime` ... `Background` | `Data` | Mixed-criticality systems |
| CongestionControl | `Drop`, `Block` | `Drop` | Disposable vs. critical data |
| Express | `true`, `false` | `false` | Latency-sensitive data |
