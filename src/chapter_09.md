# Data Serialization

This chapter covers how Zenoh handles payload bytes: the `ZBytes` container, built-in conversions, the `deserialize` pattern, and encoding metadata.

## ZBytes

`ZBytes` is Zenoh's owned byte container. All payloads — puts, subscriber samples, queryable replies — carry a `ZBytes`. It stores a sequence of bytes without imposing a schema.

### What it is

`ZBytes` wraps an internal sequence of byte slices. It owns its data and implements `Send + Sync`, making it safe to move across async task boundaries. It has no inherent schema or type information; that concern belongs to `Encoding` (covered below) and to the application.

### When to use it

Every time you send or receive data in Zenoh, you interact with `ZBytes`. You create one when publishing, and you consume one when receiving a sample or reply.

### How to use it

Creating `ZBytes` from common types via `From`:

```rust
use zenoh::bytes::ZBytes;

let b: ZBytes = "hello".into();
let b: ZBytes = 42i32.into();
let b: ZBytes = 3.14f64.into();
let b: ZBytes = vec![1u8, 2, 3].into();
```

Converting back:

```rust
// To string
let s = zbytes.try_to_string().unwrap_or_else(|e| e.to_string().into());

// To typed value via deserialize
let n: i32 = zbytes.deserialize().unwrap();

// To raw bytes
let v: Vec<u8> = zbytes.into();
```

### What to watch out for

`try_to_string` returns a `Cow<str>`. If the bytes are valid UTF-8, it borrows from the internal buffer. If not, it returns an error, which you must handle explicitly. Do not use `unwrap()` blindly on untrusted data.

---

## Primitives and ZBytes

`ZBytes` supports direct conversions for all primitive Rust types. The `From` and `Into` implementations cover `bool`, `u8` through `u128`, `i8` through `i128`, `f32`, and `f64`. The serialized format is the native little-endian representation of the type.

### When to use it

Use primitive conversions when you control both the publisher and subscriber and both run Zenoh Rust. If interoperability with other languages or serialization formats is required, use a structured format (see the section on structured types below).

### How to use it

Publisher side, following the `z_put_float.rs` pattern:

```rust
use zenoh::{bytes::Encoding, Config};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    // Float
    let value: f64 = 3.14159;
    session
        .put("demo/sensors/temperature", value)
        .encoding(Encoding::APPLICATION_FLOAT64)
        .await
        .unwrap();

    // Integer
    let counter: i64 = 42;
    session
        .put("demo/counter", counter)
        .encoding(Encoding::APPLICATION_INTEGER)
        .await
        .unwrap();
}
```

Subscriber side, deserializing by type:

```rust
while let Ok(sample) = subscriber.recv_async().await {
    let temperature: f64 = sample.payload().deserialize().unwrap();
    println!("Temperature: {temperature}");
}
```

### What to watch out for

The `deserialize` method is type-directed: it infers the target type from context. Ensure the type on the subscriber matches the type used by the publisher. Mismatched sizes (e.g., receiving as `f32` what was sent as `f64`) produce a deserialization error, not a silent truncation.

---

## Encoding

The `Encoding` struct conveys the format of the payload. It is metadata — Zenoh does not enforce it, but applications use it to select the right deserializer.

### What it is

`Encoding` is a string-based identifier attached to a publication or reply. It is carried in the message header and available on received samples via `sample.encoding()`. It has no effect on routing or delivery; it is purely advisory.

### When to use it

Set encoding whenever the payload format is not self-describing. This allows subscribers to inspect the encoding before attempting deserialization and to reject or convert formats they do not recognize.

### How to use it

Standard constants:

| Constant | Meaning |
|---|---|
| `Encoding::TEXT_PLAIN` | UTF-8 text |
| `Encoding::APPLICATION_JSON` | JSON |
| `Encoding::APPLICATION_OCTET_STREAM` | raw bytes |
| `Encoding::APPLICATION_INTEGER` | integer (little-endian native) |
| `Encoding::APPLICATION_FLOAT64` | IEEE 754 64-bit float |
| `Encoding::APPLICATION_PROPERTIES` | key=value pairs |

Custom encodings for application-specific formats:

```rust
use zenoh::bytes::Encoding;

let enc = Encoding::new("application/protobuf");
session
    .put("demo/proto", serialized_bytes)
    .encoding(enc)
    .await
    .unwrap();
```

On the receive side:

```rust
while let Ok(sample) = subscriber.recv_async().await {
    let enc = sample.encoding();
    println!("Received with encoding: {enc}");
}
```

### What to watch out for

Encoding is not enforced. A publisher can declare `APPLICATION_JSON` and send binary garbage; Zenoh will not reject it. Validation is the application's responsibility. Treat the encoding field as a hint, not a guarantee, when receiving from untrusted publishers.

---

## Iterating Bytes

`ZBytes` can hold a sequence of contiguous byte slices internally. Direct iteration over those slices avoids a copy when you only need to read the bytes.

### When to use it

Use slice iteration when passing the raw bytes directly to a parser (e.g., a protobuf decoder) that accepts `&[u8]` and you want to avoid a heap allocation. For most cases, collecting to `Vec<u8>` is simpler and sufficient.

### How to use it

Iterate the raw slices:

```rust
for slice in zbytes.slices() {
    // process raw &[u8]
}
```

Collect to `Vec<u8>`:

```rust
let bytes: Vec<u8> = zbytes.into();
```

### What to watch out for

The `slices()` iterator yields `&[u8]` references that borrow from `ZBytes`. The `ZBytes` value must remain alive for the duration of the iteration. Moving or dropping it while iterating is a compile error. If you need to send the bytes across an await point, collect them first.

---

## Structured Types with z_formats

Zenoh provides a `z_serialize`/`z_deserialize` pattern for types that implement `Serialize`/`Deserialize` from `serde`. For JSON payloads, `serde_json` is the standard choice. This maps to the `z_formats.rs` example.

### What it is

Rather than manually converting to and from `Vec<u8>`, you serialize your domain types directly to a byte representation and attach the appropriate encoding. The subscriber reverses the process.

### When to use it

Use structured serialization when your payloads carry typed, multi-field data: sensor readings, commands, status messages. Prefer JSON for interoperability with non-Rust systems. Prefer a compact binary format (protobuf, CBOR, MessagePack) for bandwidth-constrained links.

### How to use it

Add dependencies to `Cargo.toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
zenoh = "1"
tokio = { version = "1", features = ["full"] }
```

Publisher:

```rust
use serde::{Deserialize, Serialize};
use zenoh::Config;

#[derive(Serialize, Deserialize, Debug)]
struct SensorReading {
    temperature: f64,
    humidity: f64,
}

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let reading = SensorReading {
        temperature: 23.5,
        humidity: 60.0,
    };
    let payload = serde_json::to_vec(&reading).unwrap();

    session
        .put("demo/sensor/reading", payload)
        .encoding(zenoh::bytes::Encoding::APPLICATION_JSON)
        .await
        .unwrap();
}
```

Subscriber side:

```rust
while let Ok(sample) = subscriber.recv_async().await {
    let bytes: Vec<u8> = sample.payload().into();
    let reading: SensorReading = serde_json::from_slice(&bytes).unwrap();
    println!("{reading:?}");
}
```

### What to watch out for

`serde_json::to_vec` allocates. For high-frequency small messages, consider pre-allocating a buffer with `serde_json::to_writer` targeting a `Vec<u8>` that you reuse across iterations. Also consider that `unwrap()` on deserialization will panic if the remote publisher sends a malformed payload; use `match` or `?` in production code.

---

## Attachments

Samples and queries can carry an attachment: a secondary `ZBytes` for metadata. Attachments are separate from the main payload and are not interpreted by the routing layer. Common uses are correlation IDs, distributed trace context, and sequence numbers.

### What it is

An attachment is an optional `ZBytes` carried alongside the primary payload. It is available on received samples via `sample.attachment()` and on received queries via `query.attachment()`.

### When to use it

Use attachments when you need to carry out-of-band metadata without modifying the primary payload schema. This is preferable to embedding metadata fields in the payload itself when the payload schema is defined externally (e.g., a protobuf schema you do not control).

### How to use it

Publishing with an attachment:

```rust
publisher
    .put("data")
    .attachment("trace-id-1234")
    .await
    .unwrap();
```

Receiving the attachment:

```rust
if let Some(att) = sample.attachment() {
    let id = att.try_to_string().unwrap_or_else(|e| e.to_string().into());
    println!("attachment: {id}");
}
```

Attachments on queries (from the queryable side):

```rust
while let Ok(query) = queryable.recv_async().await {
    if let Some(att) = query.attachment() {
        let meta = att.try_to_string().unwrap_or_else(|e| e.to_string().into());
        println!("query attachment: {meta}");
    }
    query.reply(query.key_expr(), "response").await.unwrap();
}
```

### What to watch out for

Attachments are not indexed or filtered by the router. You cannot route or filter by attachment content. If you need to route on metadata, encode it in the key expression instead.
