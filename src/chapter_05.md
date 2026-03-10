# Publishers and Put

This chapter covers publishing data: one-shot `put`, the `delete` operation, and declared publishers for repeated writes.

## One-Shot Put

`session.put(key_expr, payload)` publishes a single value to the network and returns. No resource is pre-allocated before the call. This is appropriate for infrequent writes where per-call setup overhead is acceptable.

**What it is:** A single publish operation that sends one sample to all matching subscribers and storages.

**When to use it:** Use `put` when writes are infrequent — configuration updates, one-time events, or scripts that publish once and exit.

**How to use it:**

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    session.put("demo/example/zenoh-rs-put", "Put from Rust").await.unwrap();
}
```

You can attach encoding metadata to the payload with `.encoding()`. The encoding is informational — zenoh does not transcode data. Receivers can use it to select a deserializer.

```rust
use zenoh::{bytes::Encoding, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    session
        .put("demo/temperature", "23.5")
        .encoding(Encoding::TEXT_PLAIN)
        .await
        .unwrap();
}
```

**What to watch out for:** Each call to `session.put()` performs routing resolution on every invocation. If your publish rate exceeds a few Hz, declare a publisher instead to avoid repeated overhead.

## Delete

`session.delete(key_expr)` publishes a `Sample` with `kind = SampleKind::Delete`. Subscribers receive the sample and can act on it. Storages remove the entry for that key. The key is not required to have a prior `Put` — a delete on an unknown key is well-defined and simply results in a no-op at storages that do not hold it.

**What it is:** A publish operation that signals removal of a key rather than the presence of a value.

**When to use it:** Use `delete` when you want to signal that a resource no longer exists — for example, when a device goes offline or a configuration entry is removed.

**How to use it:**

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    session.delete("demo/example/zenoh-rs-put").await.unwrap();
    session.close().await.unwrap();
}
```

**What to watch out for:** Subscribers that do not inspect `sample.kind()` will receive delete samples as regular deliveries. If your subscriber only handles put data, check the kind before accessing the payload, since a delete sample carries no meaningful payload.

## Declared Publishers

`session.declare_publisher(key_expr)` pre-declares the publisher with the routing fabric. The routing path is negotiated once at declaration time. Subsequent writes use the pre-established path, eliminating per-write routing resolution. Use declared publishers for high-frequency writes.

**What it is:** A publisher handle bound to a fixed key expression, with routing pre-negotiated at declaration time.

**When to use it:** Use a declared publisher when writing at more than a few Hz, when you want publisher-side QoS options, or when you want the routing fabric to pre-establish paths before the first write.

**How to use it:**

```rust
use std::time::Duration;
use zenoh::{bytes::Encoding, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("demo/example/zenoh-rs-pub")
        .await
        .unwrap();

    for idx in 0..u32::MAX {
        tokio::time::sleep(Duration::from_secs(1)).await;
        let buf = format!("[{idx:4}] Pub from Rust");
        publisher
            .put(buf)
            .encoding(Encoding::TEXT_PLAIN)
            .await
            .unwrap();
    }
}
```

**What to watch out for:** The declared publisher is bound to a single key expression. It does not support wildcard key expressions in the key it publishes to — each `put` call on the publisher sends to the exact key declared. The publisher handle must remain in scope for the duration of use; dropping it undeclares the publisher and releases routing resources.

## Publisher Options

QoS options are set at declaration time via builder methods. They apply to every `put` call made through that publisher.

**What it is:** Per-publisher quality-of-service settings that control delivery semantics.

**When to use it:** Set these when the default best-effort, drop-on-congestion behavior is not acceptable for your use case.

**How to use it:**

- `reliability(Reliability::Reliable)` — acknowledged delivery; the default is `BestEffort`
- `priority(Priority::RealTime)` — one of seven priority levels from `RealTime` down to `Background`
- `congestion_control(CongestionControl::Block)` — block the publisher when the network is congested; the default is `Drop`
- `express(true)` — bypass batching to minimize latency at the cost of throughput

```rust
use zenoh::{qos::{CongestionControl, Priority, Reliability}, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let publisher = session
        .declare_publisher("control/cmd")
        .reliability(Reliability::Reliable)
        .priority(Priority::RealTime)
        .congestion_control(CongestionControl::Block)
        .await
        .unwrap();
    publisher.put("stop").await.unwrap();
}
```

**What to watch out for:** `CongestionControl::Block` causes the `put` call to block the current async task when the network is saturated. In an async context this can delay other tasks sharing the same executor thread. Full QoS details are in chapter 10.

## Matching Listener

A declared publisher can observe whether any matching subscribers currently exist. This allows a publisher to avoid expensive serialization work when no one is listening.

**What it is:** A callback that fires whenever the set of matching subscribers changes — transitioning from zero to non-zero, or non-zero to zero.

**When to use it:** Use this to gate expensive payload construction, or to log connectivity state in long-running services.

**How to use it:**

```rust
publisher
    .matching_listener()
    .callback(|status| {
        if status.matching() {
            println!("Publisher has matching subscribers.");
        } else {
            println!("Publisher has no matching subscribers.");
        }
    })
    .background()
    .await
    .unwrap();
```

**What to watch out for:** The matching status reflects the routing fabric's view of subscriber interest. A subscriber on a remote node may take a short time to propagate through the routing layer. Do not treat the matching status as an instantaneous guarantee.

## put() vs declare_publisher()

Use `session.put()` for occasional writes where per-call setup overhead is negligible — scripts, command-line tools, or infrequent state updates. Use `session.declare_publisher()` when writing at more than a few Hz, when you want publisher-side QoS pre-configured at declaration time, or when you want the routing fabric to pre-establish paths before the first write arrives.

The declared publisher also enables the matching listener feature. One-shot `put` has no equivalent mechanism for observing subscriber presence.
