# Routing and Topology

This chapter explains how Zenoh sessions discover each other and how data flows across the network in peer, client, and router modes.

## Three Modes

Zenoh has three session modes, set via the `mode` config field:

| Mode | Discovery | Routing | Use case |
|---|---|---|---|
| `peer` | Multicast + gossip | Peers route to each other | Local network, development |
| `client` | Connects to router | Router does all routing | Constrained devices, WAN |
| `router` | Connects to other routers | Full routing table | Infrastructure nodes |

A session declares its mode at open time. The mode cannot be changed after the session is open.

---

## Peer Mode

### What it is

Peer mode is the default. Each peer participates in discovery and routes data directly to other reachable peers. No central infrastructure is required. Two peers on the same L2 segment can exchange data without any router.

### When to use it

Peer mode is suitable for LAN environments, robotics labs, and development. Use it when all participants are on the same network segment or can reach each other via TCP without NAT.

### How to use it

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    // Config::default() uses peer mode with multicast scouting.
    let session = zenoh::open(Config::default()).await.unwrap();
    println!("Running as peer");
    std::future::pending::<()>().await;
}
```

### What to watch out for

In peer mode, every peer maintains its own routing table and forwards relevant traffic to reachable peers. As the number of peers grows, the overhead of maintaining pairwise connections increases. For deployments with hundreds of nodes, consider client mode with dedicated routers to centralize the routing burden.

---

## Client Mode

### What it is

Client mode is a thin mode: the client connects to one or more routers and delegates all routing to them. The client itself does not route any data. From the router's perspective, a client is a leaf node.

### When to use it

Use client mode for constrained nodes with limited CPU or memory, mobile nodes that move between networks, and nodes behind NAT where peer-to-peer connectivity is not possible. A client needs only a TCP connection to a known router address.

### How to use it

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let mut config = Config::default();
    config.insert_json5("mode", r#""client""#).unwrap();
    config
        .insert_json5("connect/endpoints", r#"["tcp/192.168.1.10:7447"]"#)
        .unwrap();
    let session = zenoh::open(config).await.unwrap();
    println!("Running as client");
    std::future::pending::<()>().await;
}
```

### What to watch out for

If all configured routers are unreachable, the client session fails to open. In production, configure multiple router endpoints for redundancy. Client sessions do not re-route through alternative peers if the router goes down; the session must be re-opened with a different endpoint.

---

## Router Mode and zenohd

### What it is

`zenohd` is the reference router implementation. It runs as a daemon and provides full routing table management, subscription aggregation, and inter-router connectivity. Routers form a mesh: each connects to others and shares routing tables. They route data between clients and peers that cannot reach each other directly.

### When to use it

Deploy routers as infrastructure nodes at network boundaries, in datacenters, and as entry points for WAN-connected clients. A router is required when clients on different L2 segments need to exchange data.

### How to use it

Run the router with default settings:

```bash
zenohd
```

Run with a config file:

```bash
zenohd -c /etc/zenoh/config.json5
```

Key config fields for a router:

```json5
{
  mode: "router",
  listen: {
    endpoints: ["tcp/0.0.0.0:7447"]
  },
  connect: {
    endpoints: ["tcp/other-router.example.com:7447"]
  }
}
```

### What to watch out for

A router that connects to no other routers is an island. Clients connected to it can only communicate with each other. For inter-region connectivity, configure explicit `connect/endpoints` to peer routers. Monitor router memory usage: a router holds subscription state for all connected clients and peers.

---

## Multicast Scouting

### What it is

In peer mode, Zenoh sends scout messages on `224.0.0.224:7446` (UDP multicast). All Zenoh peers on the same L2 segment respond with their locators, enabling automatic discovery without any prior configuration.

### When to use it

Multicast scouting is appropriate for development environments, lab networks, and any deployment where all participants share an L2 segment and multicast is permitted. It requires no configuration: `Config::default()` enables it.

### How to use it

Multicast scouting is active by default. To disable it for production environments or firewalled networks:

```rust
let mut config = Config::default();
config
    .insert_json5("scouting/multicast/enabled", "false")
    .unwrap();
```

### What to watch out for

Many cloud and datacenter networks block multicast. If peers fail to discover each other, verify that `224.0.0.224` is reachable on the local segment. On Docker or Kubernetes networks, multicast is typically unavailable; use static peer configuration instead (see the section below).

---

## Gossip Scouting

### What it is

When multicast is unavailable, Zenoh routers gossip peer information to connected clients. A client connecting to a router learns about other peers indirectly through the router's advertisement of known locators. Gossip scouting is an alternative discovery mechanism for networks where multicast is blocked.

### When to use it

Enable gossip scouting in production environments where multicast is not available but you still want dynamic peer discovery mediated by routers.

### How to use it

Router-side configuration:

```json5
{
  scouting: {
    gossip: {
      enabled: true
    }
  }
}
```

### What to watch out for

Gossip discovery adds latency relative to multicast: a client must first connect to a router before learning about other peers. In time-sensitive startup scenarios, supplement gossip with static peer endpoints so that critical connections are established before gossip converges.

---

## Static Peers

### What it is

Static peer configuration bypasses discovery entirely. The session connects directly to the listed endpoints at startup. No scout messages are sent or processed for those endpoints.

### When to use it

Use static configuration in datacenter and production environments where addresses are known, stable, and multicast is unavailable. Static configuration is also appropriate when you need deterministic startup behavior without waiting for scouting to converge.

### How to use it

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let mut config = Config::default();
    config
        .insert_json5("scouting/multicast/enabled", "false")
        .unwrap();
    config
        .insert_json5(
            "connect/endpoints",
            r#"["tcp/10.0.0.1:7447", "tcp/10.0.0.2:7447"]"#,
        )
        .unwrap();
    let session = zenoh::open(config).await.unwrap();
    std::future::pending::<()>().await;
}
```

### What to watch out for

If none of the configured endpoints are reachable, the session opens in a degraded state or fails, depending on configuration. For peer mode with static endpoints, the session proceeds even if some endpoints are down, but those peers are not reachable. Test your failover behavior explicitly.

---

## z_scout: Discovering the Network

### What it is

`zenoh::scout()` sends scouting messages and reports discovered peers and routers without opening a full session. It returns an asynchronous receiver of `Hello` messages.

### When to use it

Use `scout` for network diagnostics, topology inspection, and tooling that needs to enumerate the network without joining it as a participant.

### How to use it

```rust
use zenoh::{config::WhatAmI, scout, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let receiver = scout(WhatAmI::Peer | WhatAmI::Router, Config::default())
        .await
        .unwrap();

    let _ = tokio::time::timeout(std::time::Duration::from_secs(1), async {
        while let Ok(hello) = receiver.recv_async().await {
            println!("{hello}");
        }
    })
    .await;

    receiver.stop();
}
```

### What to watch out for

`scout` uses the same multicast address as normal scouting. In environments where multicast is disabled, it discovers nothing. The `timeout` wrapper is important: without it, `recv_async` blocks indefinitely waiting for responses that will not arrive. Always call `receiver.stop()` when done to release the multicast socket.

---

## Regionalization

### What it is

Large deployments partition the network into regions. Each region has local routers that handle intra-region traffic. Inter-region traffic flows through designated backbone routers that connect the regions. This limits the blast radius of routing updates and subscription state to the relevant region.

### When to use it

Use regionalization in deployments with hundreds or thousands of nodes, particularly when nodes in different geographic locations or administrative domains should be isolated from each other's traffic by default.

### How to use it

Key expressions carry the region scope by convention. A common pattern is to prefix all key expressions with a region identifier:

```
region/europe/sensors/temperature
region/asia/sensors/temperature
```

Backbone routers are configured to forward only subscriptions that match inter-region patterns. Intra-region traffic stays within the region's router mesh. The routing fabric forwards only relevant subscriptions across region boundaries based on declared interests.

### What to watch out for

Regionalization is a configuration and naming discipline, not an automatic feature. Misconfigured backbone router routing rules can leak intra-region traffic across regions or silently black-hole inter-region subscriptions. Test cross-region delivery explicitly in your staging environment.

---

## Firewall Considerations

Zenoh uses the following ports by default. Ensure they are open on any firewall between communicating nodes:

- Multicast UDP scouting: `224.0.0.224:7446`
- Zenoh TCP transport: port `7447`
- Zenoh TLS transport: port `7448`

For routers on public IP addresses, use explicit endpoint configuration and disable multicast:

```json5
{
  mode: "router",
  listen: {
    endpoints: ["tls/0.0.0.0:7448"]
  },
  scouting: {
    multicast: {
      enabled: false
    }
  }
}
```

TLS is recommended for any router exposed on a public network. See the TLS configuration documentation for certificate setup.
