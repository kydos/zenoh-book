# Real-World Applications

This chapter covers integrating Zenoh into production systems: embedded Rust runtimes, ROS 2, V2X, edge-to-cloud telemetry, and industrial IoT.

## Embedding Zenoh in an Application

The `Session` type is `Clone` and holds an internal `Arc`. Cloning a session is cheap: all clones share the same underlying connection state. Pass clones to independent tasks rather than wrapping the session in an additional `Arc<Mutex<_>>`.

```rust
use std::sync::Arc;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = Arc::new(zenoh::open(Config::default()).await.unwrap());

    let s1 = session.clone();
    let s2 = session.clone();

    let task1 = tokio::spawn(async move {
        s1.put("app/status", "running").await.unwrap();
    });

    let task2 = tokio::spawn(async move {
        let subscriber = s2.declare_subscriber("app/**").await.unwrap();
        while let Ok(sample) = subscriber.recv_async().await {
            println!("{}", sample.key_expr().as_str());
        }
    });

    let _ = tokio::join!(task1, task2);
}
```

Declare subscribers and publishers once at startup and reuse them. Declaring and dropping them in a loop is expensive: each declaration involves a network announcement.

## Channel Integration

Route Zenoh samples into business logic via a Tokio channel. The callback-based subscriber allows bridging from the Zenoh callback thread to an async processing loop.

```rust
use tokio::sync::mpsc;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let (tx, mut rx) = mpsc::channel::<String>(32);

    let _sub = session
        .declare_subscriber("service/input")
        .callback(move |sample| {
            let msg = sample
                .payload()
                .try_to_string()
                .unwrap_or_else(|e| e.to_string().into())
                .to_string();
            let _ = tx.blocking_send(msg);
        })
        .background()
        .await
        .unwrap();

    while let Some(msg) = rx.recv().await {
        println!("Processing: {msg}");
        // business logic
    }
}
```

The `.background()` call keeps the subscriber alive for the lifetime of the session rather than tying it to a local variable. The channel backpressure (channel capacity 32) signals the callback thread to slow down if the business logic loop falls behind.

## Graceful Shutdown

Use `tokio::signal::ctrl_c` with `tokio::select` to stop the application cleanly and flush the session:

```rust
use tokio::signal;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let subscriber = session.declare_subscriber("demo/**").await.unwrap();

    tokio::select! {
        _ = signal::ctrl_c() => {
            println!("Shutting down.");
        }
        result = async {
            while let Ok(sample) = subscriber.recv_async().await {
                println!("{}", sample.key_expr().as_str());
            }
        } => {
            let _ = result;
        }
    }

    session.close().await.unwrap();
}
```

Call `session.close().await` explicitly. This flushes pending puts and undeclares subscribers and publishers at the network level, allowing peers to update their routing tables promptly.

## ROS 2: rmw_zenoh

`rmw_zenoh` is the Zenoh-based ROS Middleware implementation. As of ROS 2 Kilted Kaiju (2025), rmw_zenoh is a Tier-1 supported RMW. It replaces rmw_fastrtps as the default for new deployments that prioritize WAN, cross-LAN, and embedded support.

Key expression conventions in rmw_zenoh:

- Topics map to key expressions under `ros2_<namespace>/<topic>`
- Services map to `ros2_<namespace>/<service_name>/request` and `.../response`
- Actions use the same pattern with `/goal`, `/result`, `/feedback`

Set `RMW_IMPLEMENTATION=rmw_zenoh_cpp` before launching ROS 2 nodes. The Zenoh session config is read from `$ZENOH_CONFIG` or the compiled-in default config.

Benefits over DDS in ROS 2:

- No multicast flooding at scale: Zenoh uses gossip-based discovery
- WAN bridging without DDS bridges or VPNs
- SHM for intra-host camera and LiDAR traffic
- Smaller wire overhead for embedded nodes

## V2X: Vehicle-to-Everything

Key expression patterns for vehicular communication:

```
v2x/<region>/<vehicle-id>/telemetry    — vehicle publishes position, speed
v2x/<region>/rsu/<rsu-id>/signal       — RSU publishes signal phase data
v2x/<region>/<vehicle-id>/alert        — roadside alerts to specific vehicle
v2x/<region>/**/hazard                 — broadcast hazard alerts
```

Vehicles declare liveliness tokens on `v2x/<region>/<vehicle-id>/presence`. A roadside unit monitors fleet density by subscribing to `v2x/<region>/**/presence` and tracking token appearances and expirations.

The liveliness pattern provides implicit fleet tracking without any application-level heartbeat protocol: Zenoh handles token expiration when a vehicle disconnects.

## Edge-to-Cloud Telemetry Pipeline

Edge devices publish sensor data. A regional edge gateway stores and aggregates it. The cloud issues queries to pull current readings.

```
[Sensor Node] → put("plant/<id>/temp", value)
     |
     v
[Edge Gateway] — storage subscriber, retains last N samples per key
     |
     v
[Cloud] → get("plant/**/temp") with ConsolidationMode::Latest
```

The cloud uses `ConsolidationMode::Latest` to receive the most recent reading per sensor without duplicates when multiple gateways hold the same key:

```rust
use zenoh::{query::ConsolidationMode, Config};

let replies = session
    .get("plant/**/temp")
    .consolidation(ConsolidationMode::Latest)
    .await
    .unwrap();

while let Ok(reply) = replies.recv_async().await {
    if let Ok(sample) = reply.result() {
        println!("{}: {}", sample.key_expr().as_str(), sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
    }
}
```

## Industrial IoT: Control Loops with SHM

In a soft-PLC or motion controller pattern, three tasks communicate through Zenoh topics:

1. A sensing task reads hardware and publishes values to SHM-backed topics.
2. A control algorithm subscribes, reads the latest value via `RingChannel(1)`, and computes actuation.
3. An actuation task subscribes to the output and drives hardware.

`RingChannel(1)` ensures the control algorithm always processes the most recent sample rather than accumulating a queue of stale readings.

```rust
use zenoh::{handlers::RingChannel, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    // Control algorithm: always use the latest sensor value
    let sensor_sub = session
        .declare_subscriber("machine/axis/position")
        .with(RingChannel::new(1))
        .await
        .unwrap();

    let actuator_pub = session
        .declare_publisher("machine/axis/setpoint")
        .express(true)
        .await
        .unwrap();

    loop {
        if let Ok(sample) = sensor_sub.recv_async().await {
            let pos: f64 = sample.payload().deserialize().unwrap_or(0.0);
            let setpoint = compute_setpoint(pos);
            actuator_pub.put(setpoint).await.unwrap();
        }
    }
}

fn compute_setpoint(pos: f64) -> f64 {
    // PID or other control law
    -pos * 0.1
}
```

`express(true)` on the actuator publisher bypasses batching so that setpoint updates reach the actuation task with minimal added latency. Combined with SHM transport on the sensing side, the full sensing-to-actuation path involves no kernel copies and no batching delays.
