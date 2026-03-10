# Debugging and Logging

This chapter covers diagnosing Zenoh applications: log levels, programmatic logging, session inspection, wire tracing, and common failure modes.

## Log Levels

Zenoh uses the `tracing` crate. Set the log level with the `ZENOH_LOG` environment variable:

```bash
ZENOH_LOG=debug cargo run
ZENOH_LOG=trace cargo run
```

Levels in increasing verbosity: `error`, `warn`, `info`, `debug`, `trace`.

| Level | What it covers |
|-------|----------------|
| `error` | Fatal errors only |
| `warn` | Recoverable errors and important warnings |
| `info` | Session open/close, discovery events |
| `debug` | Message routing, subscription matching |
| `trace` | Per-message wire events |

Start with `info` when debugging connectivity problems. Move to `debug` for routing or subscription matching issues. Use `trace` only when you need per-message wire-level events, as the output volume is high.

## Programmatic Logging Init

Call `init_log_from_env_or` before opening a session. The argument is the fallback level used when `ZENOH_LOG` is not set:

```rust
zenoh::init_log_from_env_or("error");
```

For `info` as the default:

```rust
zenoh::init_log_from_env_or("info");
```

Do not call this function twice in the same process. If your application initializes its own `tracing` subscriber, omit this call and configure filtering for the `zenoh` target directly in your subscriber setup.

## Session Inspection

`session.info()` returns the session's own identity and its known peers and routers. Use this to verify that discovery has succeeded.

```rust
use zenoh::{session::ZenohId, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let info = session.info();
    let zid: ZenohId = info.zid().await;
    let routers: Vec<ZenohId> = info.routers_zid().await.collect();
    let peers: Vec<ZenohId> = info.peers_zid().await.collect();

    println!("ZID:     {zid}");
    println!("Routers: {routers:?}");
    println!("Peers:   {peers:?}");
}
```

If `routers` and `peers` are both empty, the session has not discovered any other Zenoh node. Check multicast reachability or static peer configuration.

## Wire Tracing

For SHM diagnostics, set `ZENOH_TRACE_SHM=1`:

```bash
ZENOH_TRACE_SHM=1 ZENOH_LOG=trace cargo run
```

This emits events for SHM buffer allocation, mapping, and release. Use it when SHM throughput is lower than expected or when `alloc()` blocks unexpectedly.

## Admin Space Inspection

Query the admin space to inspect router state at runtime without modifying the application. The admin space exposes router configuration, connected peers, and active subscriptions as key-value entries.

```rust
let replies = session.get("@/router/**").await.unwrap();
while let Ok(reply) = replies.recv_async().await {
    if let Ok(sample) = reply.result() {
        let v = sample
            .payload()
            .try_to_string()
            .unwrap_or_else(|e| e.to_string().into());
        println!("{}: {}", sample.key_expr().as_str(), v);
    }
}
```

The returned values are JSON5 fragments. Parse them or pipe to `jq` for readable output.

## Running zenohd for Debugging

Start a local router with admin space read access enabled to inspect routing without modifying your application:

```bash
zenohd --cfg 'adminspace/permissions/read:true'
```

Then connect your application in client mode and query the admin space. This lets you inspect active subscriptions and routing tables without instrumenting the application code.

## Common Failure Modes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Publisher and subscriber do not communicate | UDP multicast blocked | Open 224.0.0.224:7446/UDP or use static peers |
| `session.get()` returns no replies | No matching queryable or storage | Check key expression intersection |
| High latency despite SHM | `shared-memory` feature not enabled | Add `features = ["shared-memory"]` to Cargo.toml |
| Session drops after ~30s | Firewall drops idle TCP connections | Enable keepalive or use `tcp/ip:port?tcp_keepalive=true` |
| `declare_publisher` fails | Invalid key expression | Key expressions must be in valid canonical form |
| SHM `alloc` blocks indefinitely | Provider pool exhausted | Increase provider size; reduce held buffer lifetime |

## Interpreting Debug Output

A normal session open at `info` level produces output similar to:

```
[zenoh::net::runtime] Opening session...
[zenoh::net::runtime::orchestrator] Scouting...
[zenoh::net::runtime::orchestrator] Found peer: <ZID>
[zenoh::net::runtime] Session opened with ZID: <ZID>
```

If scouting finds nothing, check multicast network reachability:

```bash
ZENOH_LOG=debug cargo run 2>&1 | grep -i scout
```

A "Scouting" line with no "Found peer" line that follows indicates that multicast is not reaching other Zenoh nodes. Either enable multicast on the network interface, or configure a static connect endpoint in the session config:

```json5
{
  connect: {
    endpoints: ["tcp/192.168.1.10:7447"]
  }
}
```
