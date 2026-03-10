# The Zenoh Data Model

This chapter defines the data structures and naming conventions that underlie every Zenoh operation: keys, key expressions, selectors, payloads, encodings, timestamps, and samples.

## Keys

A key is a `/`-separated sequence of UTF-8 path segments that uniquely identifies a resource in the Zenoh keyspace. Examples:

```
robot1/sensors/lidar
plant/zone3/temperature
vehicle/fleet/truck42/gps/position
```

Formation rules:

- No leading slash: `sensors/lidar`, not `/sensors/lidar`.
- No trailing slash: `sensors/lidar`, not `sensors/lidar/`.
- No empty segments: `sensors//lidar` is invalid.
- Segments cannot contain `*`, `$`, `?`, or `#`. Those characters are reserved for key expressions and selectors.

A key is used in write operations (`put`, `delete`) and as the identity of a stored resource. It represents a single, concrete resource — not a pattern.

## Key Expressions

A key expression (KE) is a string that matches one or more keys. Key expressions are used when declaring subscribers, queryables, and publishers, and when issuing `get` queries. A plain key with no wildcards is also a valid key expression that matches exactly itself.

### Wildcard Forms

**`*` — single-segment wildcard**

Matches exactly one non-empty path segment. The segment may contain any character except `/`.

```
robot1/sensors/*
```

Matches: `robot1/sensors/lidar`, `robot1/sensors/camera`, `robot1/sensors/imu`

Does not match: `robot1/sensors/lidar/scan` (two segments where one is expected)

**`$*` — substring wildcard**

Matches any substring within a single segment. It can appear at the start, end, or middle of a segment alongside literal characters.

```
robot$*/sensors
```

Matches: `robot1/sensors`, `robot42/sensors`, `robot_arm/sensors`

Does not match: `sensors` (no first segment), `robot1/motor/sensors` (extra segment)

A standalone `$*` in a segment position is equivalent to `*`.

**`**` — multi-segment wildcard**

Matches zero or more consecutive path segments, including the empty case (matching the position between two slashes, or the start/end of the path).

```
robot1/**/temperature
```

Matches: `robot1/temperature`, `robot1/sensors/temperature`, `robot1/sensors/internal/temperature`

`**` by itself matches any key.

### Canon Form

Zenoh normalises key expressions to a canonical form automatically. You do not need to apply these rules manually, but understanding them helps avoid writing redundant patterns:

- `**/**` reduces to `**` (two consecutive multi-segment wildcards are one)
- `**/*` becomes `*/**` (the single-segment wildcard floats to the left of the multi-segment)
- `$*$*` reduces to `$*` (two adjacent substring wildcards merge)
- A standalone `$*` in a full-segment position is equivalent to `*`

Attempting to use a malformed key expression — for example, one with adjacent `**` separated by a slash — will produce an error at declaration time, before any data is exchanged.

### Intersection and Inclusion

Two key expressions *intersect* if there exists at least one key that matches both. A key expression *A* *includes* key expression *B* if every key matching *B* also matches *A*. These relations are used internally by Zenoh's routing layer to determine which subscribers receive which publications, and which queryables receive which queries.

## Selectors

A selector extends a key expression with an optional query parameter string:

```
key/expression?param1=val1;param2=val2
```

The query string uses URL-query-string syntax with `;` as the separator between parameters. Selectors appear in `session.get()` calls:

```rust
let replies = session.get("robot1/sensors/**?encoding=json").await.unwrap();
```

The key expression portion is used for routing — it determines which queryables receive the query. The query string is forwarded to the queryable as-is. The queryable interprets the parameters according to its own logic; Zenoh does not enforce any meaning on the parameter names or values.

When no query string is needed, pass the key expression string directly to `session.get()`. The `?` and everything after it are optional.

## ZBytes

`ZBytes` is the universal payload container in Zenoh. Every sample carries its data as a `ZBytes` value. The type holds an arbitrary sequence of bytes and provides no built-in schema — the structure of the bytes is defined by the application and communicated via the `Encoding` field.

**Creating ZBytes**

Standard Rust types convert to `ZBytes` via `From`/`Into`:

```rust
// From a string slice
let bytes: ZBytes = "Hello, Zenoh".into();

// From a String
let s = String::from("sensor reading");
let bytes: ZBytes = s.into();

// From a byte slice
let raw: &[u8] = &[0x01, 0x02, 0x03];
let bytes: ZBytes = raw.into();
```

**Reading ZBytes**

For UTF-8 text payloads, use `try_to_string()`:

```rust
let text = sample.payload().try_to_string()
    .unwrap_or_else(|e| e.to_string().into());
println!("{text}");
```

`try_to_string()` returns a `Result<Cow<str>, _>`. On success it gives a borrowed string if the bytes are already UTF-8, or an error if they are not.

For structured data — serialisation and deserialisation with `z_serialize` and `z_deserialize` — see Chapter 9.

**What to watch out for**

`ZBytes` is not self-describing. A receiver cannot determine the structure of the payload from the bytes alone. Always pair `ZBytes` with an accurate `Encoding` value so receivers can make an informed decision about how to decode it.

## Encoding

`Encoding` is advisory metadata attached to every sample. It tells the receiver what format the payload bytes are in, using a MIME-type-like identifier. The receiver is free to ignore it, but well-behaved applications check the encoding before attempting to decode.

**Standard encoding constants**

| Constant                              | Meaning                    |
|---------------------------------------|----------------------------|
| `Encoding::TEXT_PLAIN`                | UTF-8 text                 |
| `Encoding::APPLICATION_JSON`          | JSON, encoded as UTF-8     |
| `Encoding::APPLICATION_OCTET_STREAM`  | Raw bytes, no structure    |
| `Encoding::APPLICATION_CBOR`          | CBOR binary encoding       |
| `Encoding::APPLICATION_PROTOBUF`      | Protocol Buffers           |

**Custom encodings**

```rust
use zenoh::bytes::Encoding;

let enc = Encoding::new("application/x-my-format");
```

The string is arbitrary. By convention, use `type/subtype` format following MIME conventions. Custom encodings with the same string on both sides of a session are sufficient for application-level agreement.

**Setting encoding on a put**

```rust
use zenoh::bytes::Encoding;

session
    .put("sensors/temperature", payload)
    .encoding(Encoding::APPLICATION_JSON)
    .await
    .unwrap();
```

**Reading encoding from a sample**

```rust
let enc = sample.encoding();
println!("Encoding: {enc}");
```

## Timestamps

Every Zenoh sample carries a timestamp generated by the originating session. The timestamp is a 64-bit Hybrid Logical Clock (HLC) value with approximately 3.5 nanosecond resolution, combined with the UUID of the session that created it.

The HLC advances monotonically even across clock adjustments on the originating machine, and the combination of the HLC value plus the session UUID guarantees a unique total order across the distributed system. No two samples from the same session share the same timestamp; no two samples from different sessions share the same `(timestamp, uuid)` pair.

Timestamps are used by storage plugins to perform conflict resolution: when two writes arrive at a storage node for the same key, the one with the later HLC timestamp wins.

**Accessing the timestamp**

```rust
if let Some(ts) = sample.timestamp() {
    println!("Timestamp: {ts}");
}
```

The timestamp is `Option<Timestamp>` because samples created directly by application code without going through the full Zenoh stack may lack one. In practice, samples received from a live network always carry a timestamp.

## Sample

A `Sample` is the unit of data delivery in Zenoh. It is what subscribers receive and what queryables return as replies. A sample carries:

| Field          | Access method       | Type                      |
|----------------|---------------------|---------------------------|
| Key expression | `key_expr()`        | `&KeyExpr`                |
| Payload        | `payload()`         | `&ZBytes`                 |
| Encoding       | `encoding()`        | `&Encoding`               |
| Timestamp      | `timestamp()`       | `Option<&Timestamp>`      |
| Kind           | `kind()`            | `SampleKind` (`Put` or `Delete`) |

The `kind` field distinguishes a data write (`Put`) from a deletion (`Delete`). Delete samples carry a key expression and a timestamp but no meaningful payload. Subscribers and storage nodes use the kind to determine whether to store or remove the resource.

A typical subscriber handler pattern:

```rust
while let Ok(sample) = subscriber.recv_async().await {
    match sample.kind() {
        SampleKind::Put => {
            let payload = sample.payload().try_to_string()
                .unwrap_or_else(|e| e.to_string().into());
            println!("PUT  {}: {payload}", sample.key_expr());
        }
        SampleKind::Delete => {
            println!("DEL  {}", sample.key_expr());
        }
    }
}
```

## The Admin Space

Zenoh exposes internal runtime state under the reserved `@/` key prefix. The admin space is a live, queryable view into the Zenoh infrastructure.

**What is available**

- `@/router/<router-id>/info` — identity and version of the router.
- `@/router/<router-id>/links` — active transport links and their statistics.
- `@/router/<router-id>/subscribers` — declared subscribers visible to this router.
- `@/router/<router-id>/queryables` — declared queryables visible to this router.
- `@/router/<router-id>/config/...` — runtime configuration (if write access is enabled).

**Reading admin data**

```rust
let replies = session.get("@/router/**").await.unwrap();
while let Ok(reply) = replies.recv_async().await {
    if let Ok(sample) = reply.result() {
        let payload = sample.payload().try_to_string()
            .unwrap_or_else(|e| e.to_string().into());
        println!("{}: {payload}", sample.key_expr());
    }
}
```

**Runtime configuration changes**

When the admin space is enabled with write permissions (see Chapter 4), you can PUT to `@/router/<id>/config/...` to modify router settings at runtime without restarting the daemon. Use this capability carefully in production: an incorrect config write can disrupt routing for all connected sessions.
