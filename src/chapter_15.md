# Performance and Benchmarking

This chapter covers measuring Zenoh latency and throughput using the built-in benchmark examples, and tuning the key parameters that affect performance.

## Built-in Benchmark Tools

The Zenoh repository ships four benchmark examples:

- `z_ping` / `z_pong` — round-trip latency measurement
- `z_pub_thr` / `z_sub_thr` — unidirectional throughput measurement
- `z_pub_shm_thr` — SHM throughput baseline

These are synchronous programs that use `.wait()` rather than `.await` to minimize measurement overhead. That is intentional: in a benchmark you want to measure the transport, not the async runtime scheduler.

## Latency: Ping/Pong

The pattern: `z_ping` puts a message on `test/ping`, `z_pong` subscribes and echoes it back on `test/pong`, and `z_ping` measures the round-trip time.

Run in two terminals:

```bash
# Terminal 1
cargo run --example z_pong

# Terminal 2 — 100 samples, 8-byte payload
cargo run --example z_ping -- -n 100 -s 8
```

Output format (per-sample RTT and derived latency):

```
8 bytes: seq=0 rtt=42µs lat=21µs
8 bytes: seq=1 rtt=41µs lat=20µs
```

The relevant code pattern from `z_ping.rs` — note the synchronous `.wait()` calls:

```rust
use zenoh::{bytes::ZBytes, key_expr::keyexpr, qos::CongestionControl, Wait};

fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(zenoh::Config::default()).wait().unwrap();

    let sub = session
        .declare_subscriber(keyexpr::new("test/pong").unwrap())
        .wait()
        .unwrap();
    let publisher = session
        .declare_publisher(keyexpr::new("test/ping").unwrap())
        .congestion_control(CongestionControl::Block)
        .express(true)
        .wait()
        .unwrap();

    let data: ZBytes = vec![0u8; 8].into();
    let write_time = std::time::Instant::now();
    publisher.put(data.clone()).wait().unwrap();
    let _ = sub.recv();
    let rtt = write_time.elapsed().as_micros();
    println!("rtt={rtt}µs lat={}µs", rtt / 2);
}
```

`CongestionControl::Block` and `express(true)` are set to eliminate any batching or drop-on-congestion behavior that would interfere with the latency measurement.

## Throughput: pub_thr/sub_thr

`z_pub_thr` sends messages as fast as possible. `z_sub_thr` counts received messages over a 1-second window and reports throughput in messages per second and bits per second.

```bash
# Terminal 1
cargo run --example z_sub_thr

# Terminal 2 — 8-byte payload
cargo run --example z_pub_thr -- -s 8
```

Output from `z_sub_thr`:

```
8 bytes: 1234567 msgs/s (9.9 Mbit/s)
```

Like `z_ping`, both `z_pub_thr` and `z_sub_thr` use `.wait()` for minimal runtime overhead. This is intentional for the same reason: the benchmark measures the transport path, not the async scheduler.

## SHM Throughput

`z_pub_shm_thr` is the SHM variant of `z_pub_thr`. Use it with the standard `z_sub_thr`. The `shared-memory` feature must be enabled.

```bash
cargo run --example z_pub_shm_thr --features shared-memory -- -s 8
```

Typical result: SHM throughput is 3–10x higher than TCP for large payloads (above 1 KB) because no serialization or kernel copy occurs. For small payloads the benefit is smaller due to fixed per-message overhead.

## Throughput Numbers (Reference)

These numbers are approximate and hardware-dependent. They are drawn from the Zenoh introductory paper and zenoh.io documentation.

| Transport | Payload | Throughput |
|-----------|---------|------------|
| TCP (loopback) | 64 B | ~1.5 Gbit/s |
| TCP (loopback) | 1 KB | ~3.5 Gbit/s |
| SHM | 64 B | ~4 Gbit/s |
| SHM | 1 KB | ~12 Gbit/s |

Run the benchmarks on your own hardware to establish baselines before tuning.

## Tuning Parameters

### Batch Size and Low-Latency Mode

Zenoh batches small messages before sending. The default batch size is 65535 bytes. Reducing it decreases latency at the cost of throughput. The `lowlatency` flag disables batching entirely at the transport level:

```json5
{
  transport: {
    unicast: {
      lowlatency: true
    }
  }
}
```

Use `lowlatency: true` when round-trip latency matters more than aggregate throughput, for example in control loops or interactive systems.

### Express Mode

Set `express(true)` on the publisher to bypass message batching for that publisher only, without a global config change:

```rust
let publisher = session
    .declare_publisher("test/ping")
    .express(true)
    .await
    .unwrap();
```

Express mode is the per-publisher equivalent of `lowlatency: true`. Use it for publishers that produce latency-sensitive messages alongside throughput-oriented publishers on the same session.

### CongestionControl::Block

In benchmarks, use `CongestionControl::Block` to prevent the publisher from dropping samples when the network is saturated:

```rust
let publisher = session
    .declare_publisher("test/thr")
    .congestion_control(CongestionControl::Block)
    .await
    .unwrap();
```

In production, `CongestionControl::Drop` is the default and is usually correct: dropping is safer than blocking a thread. Use `Block` only when every sample must be delivered and back-pressure is acceptable.

### Priority

For latency benchmarks, set `Priority::RealTime` to place messages at the front of the output queue. For throughput benchmarks the default `Priority::Data` is appropriate.

```rust
use zenoh::qos::Priority;

let publisher = session
    .declare_publisher("control/cmd")
    .priority(Priority::RealTime)
    .await
    .unwrap();
```

### Transport Selection

| Transport | Use when |
|-----------|----------|
| TCP | General use, reliable LAN/WAN |
| QUIC | Lossy links, cellular, WAN with NAT |
| SHM | Intra-host high-bandwidth pipelines |

Zenoh selects the best available transport automatically when both endpoints advertise the same capability. No application code change is required when the transport changes.
