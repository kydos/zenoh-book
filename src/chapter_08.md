# Queries

This chapter covers the client side of the query model: issuing queries with `session.get()`, using declared queriers for repeated queries, query targeting, consolidation modes, and reply handling.

## Issuing a Query

`session.get(selector)` sends a query to all matching queryables and storages. It returns a `Receiver<Reply>` that delivers replies asynchronously as they arrive from the network.

**What it is:** A one-shot query operation that broadcasts a request to all queryables whose key expression intersects the selector and collects their replies.

**When to use it:** Use `session.get()` for infrequent queries — startup data fetch, on-demand lookups, or administrative requests where the overhead of declaration is not justified.

**How to use it:**

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let replies = session.get("demo/example/**").await.unwrap();
    while let Ok(reply) = replies.recv_async().await {
        match reply.result() {
            Ok(sample) => {
                let payload = sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into());
                println!("'{}': '{}'", sample.key_expr().as_str(), payload);
            }
            Err(err) => {
                let payload = err.payload().try_to_string().unwrap_or_else(|e| e.to_string().into());
                println!("ERROR: '{payload}'");
            }
        }
    }
}
```

**What to watch out for:** The reply receiver closes after the query timeout expires (default 10 seconds). If you do not drain the receiver before the timeout, unconsumed replies are discarded. If your queryables are slow, set a longer timeout with `.timeout(Duration)`.

## Selectors

A selector is a key expression optionally followed by a query string: `key/expr?param1=val1;param2=val2`. The key expression part is used for routing. The parameters are delivered to queryables as `query.parameters()` and are interpreted entirely by the application.

**What it is:** The addressing mechanism for queries, extending key expressions with optional application-defined parameters.

**When to use it:** Use parameters for time-range queries, pagination, filtering hints, or any context that the queryable needs to construct the correct reply.

**How to use it:**

```rust
let replies = session
    .get("demo/data/**?start=0;end=100")
    .await
    .unwrap();
```

**What to watch out for:** Parameters are opaque to the zenoh routing layer. They are not parsed, validated, or used for routing — only the key expression part of the selector is used for that. Queryables receive the raw parameter string and must parse it themselves.

## Query with Payload

Queries can carry a payload body for RPC-style use. The queryable receives the body via `query.payload()` and uses it to compute a response.

**What it is:** A query that carries input data in its body, enabling a request-reply interaction pattern analogous to a remote procedure call.

**When to use it:** Use a payload when you need to send input data to a remote computation — function arguments, image data, configuration parameters — and receive a computed result in return.

**How to use it:**

```rust
use std::time::Duration;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let replies = session
        .get("service/compute")
        .payload("input data")
        .timeout(Duration::from_millis(5000))
        .await
        .unwrap();
    while let Ok(reply) = replies.recv_async().await {
        match reply.result() {
            Ok(s) => println!("Result: {}", s.payload().try_to_string().unwrap_or_else(|e| e.to_string().into())),
            Err(e) => println!("Error: {}", e.payload().try_to_string().unwrap_or_else(|e| e.to_string().into())),
        }
    }
}
```

**What to watch out for:** Not all queryables inspect `query.payload()`. A queryable written to handle parameter-only queries will ignore the body. Ensure the queryable and querier agree on the interaction pattern before using payloads.

## Query Targeting

`target(QueryTarget::...)` controls which queryables receive the query. The routing fabric uses the target to determine how broadly to forward the query.

**What it is:** A hint to the routing fabric about which subset of matching queryables should receive the query.

**When to use it:** Use `All` when you need replies from every matching source — for example, to aggregate sensor data from multiple nodes. Use `BestMatching` (the default) for normal lookups where one authoritative source is sufficient.

**How to use it:**

| Target | Behavior |
|--------|----------|
| `BestMatching` (default) | Route to the best matching set — typically the nearest complete queryable |
| `All` | Route to all matching queryables |
| `AllComplete` | Route to all complete queryables only |

```rust
use zenoh::{query::QueryTarget, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let replies = session
        .get("demo/example/**")
        .target(QueryTarget::All)
        .await
        .unwrap();
    while let Ok(reply) = replies.recv_async().await {
        if let Ok(sample) = reply.result() {
            println!("{}: {}", sample.key_expr().as_str(), sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
        }
    }
}
```

**What to watch out for:** `QueryTarget::All` can generate a large number of replies when many queryables match the selector. Ensure your application handles the full reply set, and consider using consolidation to reduce duplicates when querying multiple storages that hold overlapping data.

## Timeout

Queries have a default timeout of 10 seconds. After the timeout, the reply receiver is closed and no further replies arrive, even if some queryables have not yet responded.

**What it is:** A deadline after which the query is considered complete and the reply receiver is sealed.

**When to use it:** Set an explicit timeout whenever the default 10-second window is too long (interactive UIs, latency-sensitive loops) or too short (slow storage backends, high-latency links).

**How to use it:**

```rust
use std::time::Duration;

let replies = session
    .get("demo/example/**")
    .timeout(Duration::from_millis(500))
    .await
    .unwrap();
```

**What to watch out for:** A timeout that is too short causes replies from slow queryables to be dropped silently. The querier receives no notification that replies were cut off — the receiver simply closes. If completeness matters, set a timeout that is generous relative to the worst-case queryable response time.

## Consolidation

When multiple storages or queryables hold data for the same key, consolidation determines how duplicates are handled before they reach the application.

**What it is:** A policy for deduplicating replies that share the same key expression, using timestamps to determine which reply is the most recent.

**When to use it:** Use consolidation when querying multiple replicated storages to avoid processing the same logical value multiple times. Use `None` when you need every reply regardless of duplication, or when replies do not carry timestamps.

**How to use it:**

- `ConsolidationMode::None` — deliver all replies, including duplicates (default)
- `ConsolidationMode::Monotonic` — deduplicate using timestamps as replies arrive; keep the latest per key in a streaming fashion
- `ConsolidationMode::Latest` — buffer all replies, then deliver only the latest per key after the timeout

```rust
use zenoh::{query::ConsolidationMode, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let replies = session
        .get("demo/example/**")
        .consolidation(ConsolidationMode::Latest)
        .await
        .unwrap();
    while let Ok(reply) = replies.recv_async().await {
        if let Ok(sample) = reply.result() {
            println!("{}: {}", sample.key_expr().as_str(), sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
        }
    }
}
```

**What to watch out for:** `ConsolidationMode::Latest` buffers all replies until the timeout expires, then delivers the deduplicated set. The application receives no replies until the full timeout has elapsed. This is not suitable for latency-sensitive use cases. Use `Monotonic` when you want deduplication with lower latency, accepting that a later reply might supersede one you already processed.

## Declared Querier

`session.declare_querier(key_expr)` pre-declares a querier so the routing fabric can pre-establish paths to matching queryables. Use this for repeated queries to the same key expression to avoid per-query path resolution overhead.

**What it is:** A querier handle bound to a fixed key expression, with routing pre-negotiated at declaration time.

**When to use it:** Use a declared querier when issuing queries in a loop or at regular intervals to the same key expression. The per-query overhead is lower than repeated calls to `session.get()`.

**How to use it:**

```rust
use std::time::Duration;
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let querier = session
        .declare_querier("demo/example/**")
        .timeout(Duration::from_millis(5000))
        .await
        .unwrap();

    for _ in 0..10 {
        let replies = querier.get().await.unwrap();
        while let Ok(reply) = replies.recv_async().await {
            if let Ok(sample) = reply.result() {
                println!("{}: {}", sample.key_expr().as_str(), sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
            }
        }
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}
```

**What to watch out for:** The declared querier is bound to the key expression set at declaration time. To query a different key expression, declare a separate querier. As with all declared entities, the querier handle must remain in scope for the declaration to remain active.

## Reply Types

Every `Reply` delivers its outcome through `result()`.

**What it is:** The result type wrapping either a successful data sample or an error sent by the queryable.

**When to use it:** Always match on `reply.result()`. Ignoring error replies means your application silently drops failure signals from queryables.

**How to use it:**

- `Ok(Sample)` — a successful data reply; the sample carries the key, payload, encoding, and kind
- `Err(ReplyError)` — an error reply sent by the queryable via `query.reply_err()`
- A delete reply arrives as `Ok(Sample)` with `sample.kind() == SampleKind::Delete`

```rust
while let Ok(reply) = replies.recv_async().await {
    match reply.result() {
        Ok(sample) => {
            match sample.kind() {
                zenoh::sample::SampleKind::Put => {
                    println!("data: {}", sample.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
                }
                zenoh::sample::SampleKind::Delete => {
                    println!("deleted: {}", sample.key_expr().as_str());
                }
            }
        }
        Err(err) => {
            println!("queryable error: {}", err.payload().try_to_string().unwrap_or_else(|e| e.to_string().into()));
        }
    }
}
```

**What to watch out for:** The `Err` variant carries a `ReplyError` which has a `payload()` method, not a standard Rust error message. Extract the error description by calling `err.payload().try_to_string()` rather than using the `Display` or `Debug` format of `ReplyError` directly, unless the error is a simple string.

## Data Forwarding

The `zenoh_ext` crate provides a `SubscriberForward` extension that chains a subscriber directly to a publisher. This is the pattern used by `z_forward.rs`: every sample received on a source key expression is re-published on a destination key expression.

**What it is:** A utility that bridges two key expressions — receives from one, publishes to the other — without any application-level loop.

**When to use it:** Protocol bridging, namespace aliasing, and fan-out replication where no transformation is needed.

**How to use it:**

```rust
use zenoh::Config;
use zenoh_ext::SubscriberForward;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let mut subscriber = session
        .declare_subscriber("demo/example/**")
        .await
        .unwrap();
    let publisher = session
        .declare_publisher("demo/forward")
        .await
        .unwrap();

    // Forward all samples from the subscriber to the publisher.
    subscriber.forward(publisher).await.unwrap();
}
```

Add `zenoh-ext` to `Cargo.toml`:
```toml
zenoh-ext = "1.4.0"
```

**What to watch out for:** `forward()` consumes the subscriber and blocks until the subscriber channel is closed. Run it in a dedicated task if you need to do other work concurrently.
