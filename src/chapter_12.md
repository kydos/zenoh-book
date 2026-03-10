# Liveliness

This chapter covers Zenoh's liveliness protocol: declaring tokens to signal presence, subscribing to presence changes, and querying for currently alive tokens.

## What Liveliness Is

A liveliness token is a resource that exists as long as the session that declared it is alive. When the session closes — gracefully or due to failure — the token disappears and all liveliness subscribers receive a `Delete` sample for that token's key expression. This enables lightweight service registry, peer heartbeat, and presence tracking without explicit heartbeat messages.

The liveliness protocol is built into Zenoh's session layer. It does not require application-level timers or polling. The router detects session loss through transport-level keepalives and propagates the deletion event automatically.

---

## Declaring a Liveliness Token

### What it is

`session.liveliness().declare_token(key_expr)` declares a token under the given key expression. From the moment the token is declared, any liveliness subscriber matching that key expression receives a `Put` sample. The token remains active until it is explicitly undeclared or until the session that declared it closes.

### When to use it

Declare a liveliness token when your application component wants to advertise its presence on the network. The key expression should encode enough identity information for consumers to act on: service type, instance ID, region, or any other relevant attributes.

### How to use it

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let token = session
        .liveliness()
        .declare_token("group1/zenoh-rs")
        .await
        .unwrap();

    println!("Liveliness token declared. Press Ctrl-C to quit.");
    std::future::pending::<()>().await;

    // Token is automatically undeclared when dropped.
    // Explicit undeclaration:
    // token.undeclare().await.unwrap();
    drop(token);
}
```

### What to watch out for

A token is undeclared when the `LivelinessToken` value is dropped, even if no explicit `undeclare()` call is made. This is intentional: it ensures cleanup on panic or early return. However, it also means that storing the token in a local variable that goes out of scope before the application ends will silently undeclare it. Store the token in a variable whose lifetime matches the intended presence duration — typically the main function or a long-lived struct field.

---

## Subscribing to Liveliness Changes

### What it is

`session.liveliness().declare_subscriber(key_expr)` creates a subscriber that receives `Put` events when a matching token is declared and `Delete` events when a matching token disappears. With `.history(true)`, the subscriber also receives `Put` events for tokens that were already alive at the time the subscriber was declared.

### When to use it

Subscribe to liveliness changes when your application needs to react to the appearance or disappearance of other components: updating a service registry, triggering reconnection logic, alerting on unexpected departures, or building a live dashboard of active nodes.

### How to use it

```rust
use zenoh::{sample::SampleKind, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let subscriber = session
        .liveliness()
        .declare_subscriber("group1/**")
        .history(true)
        .await
        .unwrap();

    while let Ok(sample) = subscriber.recv_async().await {
        match sample.kind() {
            SampleKind::Put => println!("Alive: {}", sample.key_expr().as_str()),
            SampleKind::Delete => println!("Gone: {}", sample.key_expr().as_str()),
        }
    }
}
```

### What to watch out for

Without `.history(true)`, a subscriber only learns about tokens declared after it was created. If the subscriber starts after some tokens are already alive, it misses their initial `Put` events and has no knowledge of them until they disappear or reappear. Always use `.history(true)` when building a complete view of current presence. With `.history(true)`, the initial burst of `Put` events for pre-existing tokens arrives before any subsequent change events, but you should not assume any particular ordering within that burst.

---

## Querying Current Live Tokens

### What it is

`session.liveliness().get(key_expr)` is a one-shot query that retrieves all currently active tokens matching the key expression. It does not subscribe to future changes; it captures a point-in-time snapshot of liveness state.

### When to use it

Use `get` at startup to enumerate existing live tokens before subscribing to changes, or in diagnostic tools that need to inspect current presence without maintaining a persistent subscription. Combine `get` with a subscriber to build a complete, consistent view: first `get` the current state, then subscribe to deltas.

### How to use it

```rust
use std::time::Duration;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let replies = session
        .liveliness()
        .get("group1/**")
        .timeout(Duration::from_millis(5000))
        .await
        .unwrap();

    while let Ok(reply) = replies.recv_async().await {
        match reply.result() {
            Ok(sample) => println!("Alive: {}", sample.key_expr().as_str()),
            Err(e) => println!(
                "Error: {}",
                e.payload()
                    .try_to_string()
                    .unwrap_or_else(|e| e.to_string().into())
            ),
        }
    }
}
```

### What to watch out for

The `get` operation has a timeout. Tokens declared by sessions that are not reachable within the timeout window do not appear in the results. In large distributed deployments where routers add latency, set a generous timeout. A too-short timeout produces an incomplete snapshot that may cause the application to believe fewer services are available than actually are.

There is a race between `get` and `declare_subscriber`: tokens declared in the gap between the two calls may be missed. The correct pattern is to declare the subscriber first with `.history(true)` and then issue a `get` to populate initial state. The subscriber's history replay and the `get` results may overlap; deduplicate by key expression.

---

## Scalability Compared to DDS Liveliness

DDS liveliness requires O(n^2) heartbeats in a fully connected system: every participant must heartbeat every other participant to maintain liveness assertions. In a 1000-node system, this generates roughly 1,000,000 heartbeat messages per heartbeat period under the AUTOMATIC_LIVELINESS_QOS_POLICY.

Zenoh's liveliness protocol is O(n): each token propagates through the routing fabric once as a single event. The router detects session loss through existing transport-level keepalives and generates a single delete event per lost token. In a 1000-node system, Zenoh generates 1000 presence events per heartbeat period — three orders of magnitude fewer messages.

This matters in practice for systems that add and remove nodes frequently, such as container orchestration environments, mobile robots, and vehicular networks. The DDS approach can consume a significant fraction of available bandwidth on the monitoring channel alone; the Zenoh approach does not.

---

## Use Cases

### Service Registry

Services declare tokens on `services/<type>/<id>`. Clients subscribe with `services/<type>/**` to discover available instances of a given service type. When a service goes down, clients receive a `Delete` event and can remove it from their routing table.

### Fleet Tracking

Vehicles declare tokens on `fleet/<vehicle-id>`. A monitoring service subscribes to `fleet/**` with `.history(true)` to maintain a live roster of active vehicles. Unexpected `Delete` events trigger alerts.

### Peer Health Monitoring

Each node declares a liveliness token on `nodes/<node-id>`. A watchdog subscribes to `nodes/**` and maintains a set of expected nodes. A `Delete` event for a node that should be running triggers a remediation workflow: restart, alert, or failover.

### Leader Election

All candidates declare tokens on `election/<cluster-id>/**`. Candidates subscribe to the same prefix. The candidate whose token key expression sorts first (or by any deterministic rule applied to the key) is the leader. When the leader's session closes, all remaining candidates receive a `Delete` event and re-evaluate leadership based on the surviving tokens.

---

## Integration with Queryables

Liveliness tokens and queryables compose naturally. A service can declare both a liveliness token and a queryable on the same key expression:

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    // Advertise presence
    let _token = session
        .liveliness()
        .declare_token("services/temperature/sensor-1")
        .await
        .unwrap();

    // Serve queries
    let queryable = session
        .declare_queryable("services/temperature/sensor-1")
        .await
        .unwrap();

    while let Ok(query) = queryable.recv_async().await {
        query
            .reply(query.key_expr(), "23.5")
            .await
            .unwrap();
    }
}
```

Clients discover the service via the liveliness token and issue queries to the same key expression. When the service goes down, the liveliness token disappears and clients know not to issue further queries. This pattern avoids the need for a separate service discovery mechanism.
