# Subscribers

This chapter covers receiving data: declaring subscribers, processing samples, using wildcard key expressions, and the pull model with ring channels.

## Declaring a Subscriber

`session.declare_subscriber(key_expr)` registers interest in all puts and deletes matching the key expression. It returns a `Subscriber` handle backed by a FIFO channel by default. Incoming samples are queued and consumed by calling `recv_async()` on the handle.

**What it is:** A subscription handle that delivers all samples published to matching key expressions.

**When to use it:** Use a subscriber whenever your application needs to react to data published by other nodes — sensor readings, telemetry, commands, or any event-driven data flow.

**How to use it:**

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let subscriber = session
        .declare_subscriber("demo/example/**")
        .await
        .unwrap();

    while let Ok(sample) = subscriber.recv_async().await {
        let payload = sample
            .payload()
            .try_to_string()
            .unwrap_or_else(|e| e.to_string().into());
        println!(
            "{} {} '{}'",
            sample.kind(),
            sample.key_expr().as_str(),
            payload
        );
    }
}
```

**What to watch out for:** The loop above exits when `recv_async()` returns `Err`, which happens when the subscriber handle is dropped or the session is closed. If you hold the session and subscriber in the same scope, both must remain alive for the loop to continue running.

## Sample Fields

Each received `Sample` carries the following fields.

**What it is:** The complete unit of data delivery in zenoh — a key, a value, and metadata.

**When to use it:** Inspect sample fields whenever you need more than the payload: filtering by key, checking timestamps for ordering, or branching on put vs delete.

**How to use it:**

- `key_expr()` — the concrete key of this sample; no wildcards appear here even if the subscription used wildcards
- `payload()` — the `ZBytes` value carrying the raw bytes
- `timestamp()` — optional HLC timestamp set by the origin; `None` if the publisher did not attach one
- `encoding()` — the declared encoding; use this to select a deserializer
- `kind()` — `SampleKind::Put` or `SampleKind::Delete`
- `attachment()` — optional auxiliary bytes attached by the publisher

```rust
while let Ok(sample) = subscriber.recv_async().await {
    if let Some(ts) = sample.timestamp() {
        println!("ts={ts}");
    }
    println!("kind={} key={}", sample.kind(), sample.key_expr().as_str());
}
```

**What to watch out for:** `timestamp()` returns `None` unless the publisher explicitly set one or unless a zenoh router with timestamping enabled inserted it. Do not assume timestamps are always present.

## Wildcard Subscriptions

Key expressions in subscriptions match all publications whose key intersects the pattern. Wildcards follow the zenoh key expression syntax: `*` matches a single chunk, `**` matches any number of chunks.

**What it is:** A subscription whose key expression contains wildcards, causing it to receive samples from multiple publishers that each use concrete keys.

**When to use it:** Use wildcards when you want to receive from a class of publishers without naming each one explicitly — for example, all sensors under a prefix, or all topics at a given depth.

**How to use it:**

- `demo/example/**` receives everything under that prefix
- `*/sensors/*` receives any two-chunk key with `sensors` as the middle segment
- `demo/+/temperature` uses `+` as a single-segment wildcard (synonym for `*` in many contexts)

Wildcards in the subscription do not appear in the received `sample.key_expr()`. That field always contains the concrete key the publisher used.

**What to watch out for:** Overly broad wildcards such as `**` at the root match every key expression in the system. This is rarely what you want and can generate unexpected traffic. Use the narrowest pattern that covers your use case.

## Callback-Based Subscriber

Pass a closure with `.callback()` for fire-and-forget processing. The callback runs in the zenoh receiver thread and should complete quickly. Use `.background()` to keep the subscriber alive without holding an explicit handle.

**What it is:** A subscriber variant where incoming samples are dispatched to a user-supplied closure instead of a channel.

**When to use it:** Use callbacks when you do not need backpressure control, when the processing is lightweight, and when you want to avoid writing a receive loop.

**How to use it:**

```rust
let _subscriber = session
    .declare_subscriber("demo/example/**")
    .callback(|sample| {
        println!("received: {}", sample.key_expr().as_str());
    })
    .background()
    .await
    .unwrap();

// Keep the program alive
std::thread::park();
```

`.background()` keeps the subscriber alive as long as the session is alive, without holding an explicit handle.

**What to watch out for:** The callback runs synchronously on the receiver thread. Blocking inside the callback — for example with a mutex, a slow I/O call, or an async runtime — stalls the delivery of subsequent samples. If processing is non-trivial, hand off work to a separate task inside the callback.

## Pull Model: Ring Channel

By default the subscriber uses a FIFO. If the consumer is slower than the producer, the FIFO grows unbounded. Use `RingChannel` to cap memory and automatically drop the oldest samples when the channel is full.

**What it is:** An alternative channel handler that bounds the subscriber's internal queue and discards old entries when new ones arrive and the ring is full.

**When to use it:** Use `RingChannel` for sampling use cases where only the latest value matters — telemetry dashboards, health monitors, and other consumers that can tolerate dropping stale data.

**How to use it:**

```rust
use std::time::Duration;
use zenoh::{handlers::RingChannel, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let subscriber = session
        .declare_subscriber("demo/example/**")
        .with(RingChannel::new(3))  // keep at most 3 samples
        .await
        .unwrap();

    loop {
        match subscriber.recv_async().await {
            Ok(sample) => {
                let payload = sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into());
                println!("Pulled: {}", payload);
                tokio::time::sleep(Duration::from_secs(5)).await;
            }
            Err(e) => {
                println!("Error: {e}");
                return;
            }
        }
    }
}
```

**What to watch out for:** Samples dropped by the ring channel are silently discarded — there is no notification that a drop occurred. If your application must process every sample without loss, use the default FIFO channel and ensure the consumer keeps up with the producer.

## Subscriber Lifecycle

The subscriber is active as long as its handle is held. Dropping the handle undeclares the subscriber. Resources on remote routers are released and the routing fabric stops forwarding matching publications to this node.

**What it is:** The relationship between the Rust handle and the active subscription registered with the routing fabric.

**When to use it:** Understand the lifecycle to avoid two common mistakes: dropping the handle too early (subscription disappears) or holding it too long (resources are not released after the subscription is no longer needed).

**How to use it:**

Always keep the subscriber handle in scope for the duration you need it:

```rust
let subscriber = session.declare_subscriber("demo/data/**").await.unwrap();
// subscriber is active here
process_data(&subscriber).await;
// subscriber is dropped here; resources are released
```

Use `.background()` to tie the subscriber lifetime to the session rather than to a local variable:

```rust
session
    .declare_subscriber("demo/data/**")
    .callback(|s| println!("{}", s.key_expr().as_str()))
    .background()
    .await
    .unwrap();
// subscriber remains active until the session is closed
```

**What to watch out for:** If the subscriber handle is assigned to `_` rather than a named binding, Rust drops it immediately at the end of the statement. Assign to a named variable or use `.background()`.
