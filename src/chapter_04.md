# Sessions and Configuration

This chapter covers how to open and close a Zenoh session, how to supply configuration from a file or programmatically, and what the three operational modes mean in practice.

## Opening a Session

A Zenoh session is the top-level object through which all operations flow. Open one with `zenoh::open`:

```rust
use zenoh::Config;

let session = zenoh::open(Config::default()).await.unwrap();
```

`zenoh::open` takes a `Config` value and returns a `Result<Session, ZError>`. The `await` drives the async handshake: scouting, transport binding, and any router connections configured in the config complete before the future resolves.

The returned `Session` is an `Arc`-backed handle. Cloning it is cheap and gives another handle to the same underlying session. All clones share the same transport state, resource declarations, and lifecycle.

Drop the session to close it, or call `session.close().await` for explicit control (see the Graceful Shutdown section below).

## Default Configuration

`Config::default()` produces a configuration that:

- Sets the mode to **peer**.
- Enables multicast scouting on `udp/224.0.0.224:7446`.
- Listens for incoming connections on any available TCP port.
- Disables the admin space.

This configuration requires no setup and works immediately on a single machine or a local network where multicast is not blocked. It is the correct starting point for development and single-machine testing.

When the default configuration is not appropriate — for example, when multicast is unavailable, when you need to connect across subnets, or when you need TLS — you supply an explicit configuration.

## Configuration from File

Zenoh uses JSON5 as its configuration file format. JSON5 is a superset of JSON that allows single-line comments (`//`), trailing commas, and unquoted keys. Load a config file with:

```rust
let config = Config::from_file("path/to/config.json5").unwrap();
let session = zenoh::open(config).await.unwrap();
```

A representative config file covering the most common fields:

```json5
{
  // Operational mode: "peer", "client", or "router"
  mode: "client",

  connect: {
    // Endpoints this session connects to (client or peer mode)
    endpoints: ["tcp/192.168.1.10:7447"]
  },

  listen: {
    // Endpoints this session listens on (router or peer mode)
    endpoints: ["tcp/0.0.0.0:7447"]
  },

  adminspace: {
    enabled: true,
    permissions: {
      read: true,
      write: false   // set true to allow runtime config changes
    }
  }
}
```

**Key fields**

- `mode` — controls routing behaviour. Covered in detail in the Connection Modes section.
- `connect.endpoints` — the list of remote endpoints to dial on session open. Each endpoint is a URI string with the scheme determining the transport: `tcp/`, `udp/`, `quic/`, `tls/`.
- `listen.endpoints` — the list of local addresses to bind and accept connections on. In peer mode this is optional; the runtime picks an ephemeral port when omitted.
- `adminspace.enabled` — exposes internal state and configuration under the `@/` keyspace (see Chapter 3 for the admin space query pattern).
- `adminspace.permissions.write` — when `true`, allows PUT operations to `@/router/<id>/config/...` to change settings at runtime.

**What to watch out for**

Config file paths are resolved relative to the current working directory of the process, not relative to the config file itself. Use absolute paths in production deployments to avoid surprises when the working directory changes.

## Programmatic Configuration

When you need to construct configuration at runtime — reading from environment variables, command-line arguments, or a service registry — use `Config::default()` as a base and mutate it with `insert_json5`:

```rust
let mut config = Config::default();
config.insert_json5("mode", r#""client""#).unwrap();
config.insert_json5("connect/endpoints", r#"["tcp/127.0.0.1:7447"]"#).unwrap();
let session = zenoh::open(config).await.unwrap();
```

`insert_json5` takes a path (using `/` as a separator to navigate nested keys) and a JSON5-encoded value as a string. The value string must be valid JSON5 for the target type:

- A string value: `r#""client""#` (note the inner quotes)
- An array: `r#"["tcp/127.0.0.1:7447"]"#`
- A boolean: `"true"` or `"false"`
- A number: `"7447"`

`insert_json5` returns a `Result`. Unwrapping is fine during development; in production, propagate the error.

**Combining file and programmatic config**

You can load a base configuration from a file and then override specific fields programmatically:

```rust
let mut config = Config::from_file("base.json5").unwrap();
// Override the connect endpoint from an environment variable
let endpoint = std::env::var("ZENOH_ENDPOINT").unwrap_or_else(|_| "tcp/127.0.0.1:7447".into());
let json_endpoint = format!(r#"["{endpoint}"]"#);
config.insert_json5("connect/endpoints", &json_endpoint).unwrap();
let session = zenoh::open(config).await.unwrap();
```

## Connection Modes

### Peer

The default mode. A peer:

- Binds a local listen endpoint (TCP port, chosen automatically if not specified).
- Sends and receives multicast UDP scouting messages to discover other peers on the same network segment.
- Establishes direct unicast connections to discovered peers.
- Performs local routing: if peer A and peer B are both connected to peer C, peer C forwards messages between A and B.

Peer mode requires no pre-existing infrastructure. Two processes on the same machine in peer mode discover each other within milliseconds of starting. On a LAN, peers across machines discover each other through multicast scouting.

Peer mode is appropriate for: development, local robotics deployments, and any scenario where all participants are on the same multicast-reachable network segment.

### Client

A client:

- Does not bind any listen endpoint.
- Does not perform multicast scouting (by default).
- Connects to one or more routers specified in `connect.endpoints`.
- Delegates all routing to the connected routers.

Client mode is appropriate for: constrained devices that should not act as routing hops, processes behind NAT that cannot accept incoming connections, and any node that should be a pure consumer or producer without infrastructure responsibility.

A client session cannot communicate with another client session directly. Both must be connected to a common router.

### Router

A router:

- Binds listen endpoints and accepts connections from clients and other routers.
- Participates in router-to-router routing to forward messages across network segments.
- Can host storage plugins and admin space access.

You typically run `zenohd` rather than embedding a router in application code, but the `router` mode is available programmatically when needed (for example, in integration tests or embedded gateway applications).

## Session Info

After opening a session, `session.info()` returns a handle to runtime identity information:

```rust
use zenoh::session::ZenohId;

let info = session.info();
let zid: ZenohId = info.zid().await;
let routers: Vec<ZenohId> = info.routers_zid().await.collect();
let peers: Vec<ZenohId> = info.peers_zid().await.collect();

println!("My ZID: {zid}");
println!("Connected routers: {routers:?}");
println!("Connected peers: {peers:?}");
```

- `zid()` returns the UUID of this session. The ZID is stable for the lifetime of the session and is used as the session identifier in timestamps and admin space paths.
- `routers_zid()` returns an async stream of the ZIDs of all routers this session is currently connected to. The `.collect()` call drains the stream into a `Vec`.
- `peers_zid()` returns an async stream of peer ZIDs (in peer mode).

`session.info()` is useful for diagnostics, for logging the session identity on startup, and for constructing admin space query paths that reference a specific router.

## Runtime Configuration via Admin Space

When the admin space is open with write permissions, you can change router configuration at runtime by putting JSON5-encoded values to `@/router/<router-id>/config/<path>`:

```rust
// Enable a new listen endpoint at runtime (example only — not all fields support hot reload)
let router_zid = session.info().routers_zid().await.next().await.unwrap();
let config_key = format!("@/router/{router_zid}/config/listen/endpoints");
session
    .put(&config_key, r#"["tcp/0.0.0.0:7448"]"#)
    .await
    .unwrap();
```

Not every configuration field supports hot reload. Fields that affect transport binding or routing topology typically require a restart. Consult the Zenoh router documentation for the current list of runtime-configurable fields.

Use this capability with caution in production. An incorrect write can disrupt routing for all sessions connected to the router. Restrict write access to the admin space unless runtime reconfiguration is an explicit operational requirement.

## Graceful Shutdown

A Zenoh session closes when it is dropped. Rust's ownership model ensures the session is always closed, but "always" here means "at the point the owning variable goes out of scope." In a long-running async application, that point can be far from where you logically want the session to stop.

For explicit control, call `session.close()`:

```rust
session.close().await.unwrap();
```

`close()` consumes the session handle, performs an orderly teardown — undeclaring resources, draining send queues, and closing transport links — and returns a `Result`. Awaiting it ensures the teardown completes before your program proceeds.

**Publishers and subscribers before close**

Publishers and subscribers are cleaned up automatically when the session closes, whether by drop or by explicit `close()`. You do not need to call `undeclare()` on each entity before closing the session; the session close handles it.

If you need to undeclare a specific entity earlier in the session's lifetime — for example, to stop receiving on a subscriber without closing the whole session — call `subscriber.undeclare().await.unwrap()`. This is covered in more detail in the chapters on publishers and subscribers.

**What to watch out for**

If you hold a `Session` clone and the original is dropped, the session stays open because the `Arc` reference count is still positive. The session closes only when all clones are dropped (or when `close()` is called on any clone, which closes the underlying session for all handles).
