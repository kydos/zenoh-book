# Queryables and Storage

This chapter covers queryables — entities that reply to queries — and the storage pattern that combines a subscriber with a queryable to implement distributed key-value storage.

## What Queryables Are

A queryable is an entity that registers to answer queries on a key expression. When `session.get()` is called, the routing fabric forwards the query to all matching queryables, which then send replies back to the caller. Queryables support four patterns: data-at-rest retrieval, remote computation, RPC-style invocation, and map-reduce aggregation.

**What it is:** The server side of zenoh's query model. A queryable declares an interest in receiving queries and is responsible for generating one or more replies for each query it receives.

**When to use it:** Use a queryable whenever you want to expose data or computation that is retrieved on demand rather than continuously streamed. Unlike publishers, queryables respond only when asked.

**What to watch out for:** A queryable must call `query.reply()` or `query.reply_err()` for each received query. Failing to reply does not cause an error at the queryable, but the querier will wait until its timeout expires before closing the reply receiver.

## Declaring a Queryable

`session.declare_queryable(key_expr)` registers the queryable with the routing fabric. Returns a `Queryable` handle backed by a channel. Incoming queries are consumed with `recv_async()`.

**How to use it:**

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let key_expr = "demo/example/zenoh-rs-queryable";
    let queryable = session
        .declare_queryable(key_expr)
        .await
        .unwrap();

    while let Ok(query) = queryable.recv_async().await {
        println!("Received query: {}", query.selector());
        query
            .reply(key_expr, "Queryable from Rust")
            .await
            .unwrap_or_else(|e| println!("Reply error: {e}"));
    }
}
```

**What to watch out for:** The queryable handle must remain in scope for the queryable to be active. Dropping it undeclares the queryable and releases routing resources. Assign it to a named variable, not `_`.

## The Query Object

A `Query` carries the full context of the incoming request.

**What it is:** The incoming request object passed to a queryable, containing the key expression, optional parameters, and an optional request body.

**When to use it:** Inspect query fields to decide what data to return — filter by parameters, branch on the presence of a request body, or match against sub-keys.

**How to use it:**

The fields available on a `Query`:

- `selector()` — the full selector: key expression plus optional query string (e.g., `demo/data/**?start=0`)
- `key_expr()` — the key expression part of the selector only
- `parameters()` — the query string parameters after `?` (e.g., `start=0;end=100`)
- `payload()` — optional request body for RPC-style use; `None` if the querier sent no body
- `encoding()` — the encoding of the request body

The queryable sends replies by calling `query.reply(key, value)`. Each reply is a separate `Sample` delivered to the querier.

**What to watch out for:** `query.reply()` takes ownership of the reply value. If you need to send multiple replies from the same data, clone the value before each call.

## Query with Body (RPC Pattern)

A query can carry a payload, enabling RPC-style invocation. The queryable receives the request body via `query.payload()` and uses it to compute a response.

**What it is:** A use of the query model where the payload acts as a function argument and the reply acts as a return value.

**When to use it:** Use this pattern for remote computation — image processing, inference, aggregation — where the input data is small enough to fit in a query payload and the output is returned as one or more replies.

**How to use it:**

```rust
while let Ok(query) = queryable.recv_async().await {
    match query.payload() {
        None => println!("No request body"),
        Some(body) => {
            let req = body.try_to_string().unwrap_or_else(|e| e.to_string().into());
            println!("Request: {req}");
        }
    }
    query.reply("demo/example/result", "computed result").await.unwrap_or_else(|e| println!("Error: {e}"));
}
```

**What to watch out for:** The reply key expression must match or be contained within the queryable's declared key expression. Replying with a key that does not intersect the query key expression is not an error at the API level, but the router may drop it, and the querier may not receive it.

## Completeness

`complete(true)` declares that this queryable holds the complete set of data for its key expression. No other queryable is needed to fully answer a query on this key space. The router uses this flag to optimize query routing — it can stop at the first complete match rather than forwarding to all matching queryables.

**What it is:** A hint to the routing fabric that this queryable is the authoritative source for its declared key expression.

**When to use it:** Set `complete(true)` only when your queryable genuinely holds all current state for its declared key space. The storage pattern (described below) is the primary use case.

**How to use it:**

```rust
let queryable = session
    .declare_queryable("demo/data/**")
    .complete(true)
    .await
    .unwrap();
```

**What to watch out for:** Setting `complete(true)` incorrectly causes queries to be routed only to your queryable even when other sources hold relevant data. Use `complete(false)` (the default) when your queryable handles a subset of queries or supplements other queryables.

## Storage Pattern

A storage combines a subscriber (to receive and store incoming puts and deletes) with a queryable (to answer get requests from the stored data). This is the canonical pattern for implementing a distributed key-value store in an application.

**What it is:** A compound pattern that maintains an in-memory map of the latest sample per key and exposes that map to queriers.

**When to use it:** Use this pattern when you need application-level storage with custom logic, or when you want to store data in-process without running a separate zenohd instance with the storage plugin.

**How to use it:**

```rust
use std::collections::HashMap;
use futures::select;
use zenoh::{
    key_expr::{keyexpr, KeyExpr},
    sample::{Sample, SampleKind},
    Config,
};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();
    let key_expr = "demo/example/**";
    let mut stored: HashMap<String, Sample> = HashMap::new();

    let subscriber = session.declare_subscriber(key_expr).await.unwrap();
    let queryable = session.declare_queryable(key_expr).complete(true).await.unwrap();

    loop {
        select! {
            sample = subscriber.recv_async() => {
                let sample = sample.unwrap();
                let key = sample.key_expr().to_string();
                match sample.kind() {
                    SampleKind::Put => { stored.insert(key, sample); }
                    SampleKind::Delete => { stored.remove(&key); }
                }
            },
            query = queryable.recv_async() => {
                let query = query.unwrap();
                for (stored_key, sample) in &stored {
                    if query.key_expr().intersects(unsafe {
                        keyexpr::from_str_unchecked(stored_key)
                    }) {
                        query.reply(sample.key_expr().clone(), sample.payload().clone())
                            .await
                            .unwrap_or_else(|e| println!("Reply error: {e}"));
                    }
                }
            }
        }
    }
}
```

This pattern requires the `futures` crate for the `select!` macro.

**What to watch out for:** The `select!` macro polls both futures concurrently on a single task. If reply processing is slow — for example, when the stored map is large — incoming subscriber samples queue up during that time. For high-throughput scenarios, consider spawning separate tasks for the subscriber and the queryable rather than interleaving them with `select!`.

## Error Replies

Queryables can send error replies when processing fails. The querier receives these as `Err(ReplyError)` from `reply.result()`.

**What it is:** A mechanism for queryables to signal failure to the querier without sending a valid sample.

**When to use it:** Send an error reply when the queryable cannot fulfill the request — for example, when the requested key does not exist, when required parameters are missing, or when computation fails.

**How to use it:**

```rust
query
    .reply_err("processing failed")
    .await
    .unwrap_or_else(|e| println!("Error: {e}"));
```

The querier receives this as `reply.result() == Err(...)` (see chapter 8).

**What to watch out for:** Sending an error reply does not prevent you from also sending successful replies. A queryable can send zero or more successful replies followed by an error reply, or vice versa. The querier receives all of them in the order they arrive.

## zenohd Storage Plugin

For production storage, zenohd ships a storage plugin. Configure it in a JSON5 config file and start zenohd — no application-side code is required.

**What it is:** A router-side plugin that acts as a storage for a given key expression, answering queries from the data it has received via subscriptions.

**When to use it:** Use the zenohd storage plugin when you do not want to manage storage in your application process, when you need persistence across application restarts, or when the storage should be available independently of any particular application.

**How to use it:**

```json5
{
  plugins: {
    storage_manager: {
      storages: {
        demo: {
          key_expr: "demo/example/**",
          volume: { id: "memory" }
        }
      }
    }
  }
}
```

Start zenohd with this config: `zenohd -c config.json5`. The router then answers queries on `demo/example/**` without any application-side code.

**What to watch out for:** The `memory` volume does not persist data across zenohd restarts. For durable storage, configure a backend volume such as `rocksdb` or `influxdb`. Refer to the zenohd plugin documentation for the available backend options and their configuration keys.
