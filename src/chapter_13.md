# Shared Memory

This chapter covers Zenoh's shared memory (SHM) transport: when to use it, how to allocate SHM buffers, zero-copy publishing, and receiving SHM data.

## When SHM Applies

SHM transport is available only between processes on the same host. It provides zero-copy delivery: the publisher writes into a shared memory region, the subscriber reads directly from it without any data copy. This is the primary mechanism for high-throughput, low-latency intra-host communication in use cases such as camera frames, LiDAR point clouds, video streams, and ML inference data.

SHM is not a separate protocol — it is a transport optimization. The same publisher and subscriber API is used; Zenoh selects SHM automatically when available and configured.

The `shared-memory` feature must be enabled in `Cargo.toml`:

```toml
zenoh = { version = "1.4.0", features = ["shared-memory"] }
```

When the publisher and subscriber are on different hosts, Zenoh automatically falls back to TCP or QUIC transport. No code change is required on either side.

## Creating an SHM Provider

The SHM provider manages the shared memory region. The default backend uses POSIX shared memory. `ShmProviderBuilder::default_backend` is the simplest way to create one.

```rust
use zenoh::{
    shm::{BlockOn, GarbageCollect, ShmProviderBuilder},
    Config, Wait,
};

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    // Create a 1 MB SHM provider
    let provider = ShmProviderBuilder::default_backend(1024 * 1024)
        .wait()
        .unwrap();

    let publisher = session
        .declare_publisher("demo/shm/data")
        .await
        .unwrap();

    // Allocate a 256-byte buffer
    let mut sbuf = provider
        .alloc(256)
        .with_policy::<BlockOn<GarbageCollect>>()
        .await
        .unwrap();

    sbuf[..5].copy_from_slice(b"hello");

    // Zero-copy publish
    publisher.put(sbuf).await.unwrap();
}
```

`BlockOn<GarbageCollect>` means: if no buffer is immediately available, first run garbage collection to reclaim released buffers, then block if the pool is still exhausted. This is the recommended allocation policy for production use.

### Allocation Policies

| Policy | Behavior |
|--------|----------|
| `BlockOn<GarbageCollect>` | GC then block — recommended for production |
| `GarbageCollect` | GC then return `None` if still unavailable |
| `BlockOn` | Block immediately without GC |

Use `GarbageCollect` without blocking only in real-time contexts where blocking is prohibited.

## Receiving SHM Data

The subscriber API is identical to the non-SHM case. Zenoh handles the SHM-to-bytes mapping transparently.

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");
    let session = zenoh::open(Config::default()).await.unwrap();

    let subscriber = session
        .declare_subscriber("demo/shm/data")
        .await
        .unwrap();

    while let Ok(sample) = subscriber.recv_async().await {
        // payload may reference SHM directly — no copy occurs
        let bytes: Vec<u8> = sample.payload().into();
        println!("Received {} bytes", bytes.len());
    }
}
```

Avoid converting the payload to `Vec<u8>` in hot paths if the goal is zero-copy. Access the payload bytes directly through the `ZBytes` API to avoid materializing a copy.

## SHM Queries and Queryables

SHM works transparently with queryables. Allocate an SHM buffer inside a queryable handler and return it as the reply payload.

```rust
// Inside a queryable handler:
let mut sbuf = provider
    .alloc(1024)
    .with_policy::<BlockOn<GarbageCollect>>()
    .await
    .unwrap();
// ... fill sbuf with reply data ...
query.reply("demo/shm/data", sbuf).await.unwrap_or_else(|e| println!("Error: {e}"));
```

The querier receives the reply with the same subscriber-side transparency: no code change is needed to handle an SHM-backed reply versus a TCP-backed reply.

## Custom POSIX SHM Provider

For control over the POSIX region name and size, use `PosixShmProviderBackend` directly. This is useful when multiple providers must coexist in the same process with distinct memory regions.

```rust
use zenoh::{
    shm::{
        BlockOn, GarbageCollect, PosixShmProviderBackend, ShmProviderBuilder, POSIX_PROTOCOL_ID,
    },
    Wait,
};

let backend = PosixShmProviderBackend::builder()
    .with_size(4 * 1024 * 1024) // 4 MB
    .res()
    .wait()
    .unwrap();

let provider = ShmProviderBuilder::builder()
    .protocol_id::<POSIX_PROTOCOL_ID>()
    .backend(backend)
    .res()
    .await
    .unwrap();
```

Use this pattern when the default single-region provider is insufficient, or when you need to name the POSIX region explicitly for external tooling.

## GPU and Accelerator Backends

Zenoh's SHM provider interface is pluggable. Custom backends can wrap GPU memory (CUDA, ROCm), DMA buffers, or FPGA memory regions. The publisher-side API remains unchanged — only the provider construction differs.

This pattern is common in ML inference pipelines where the inference result lives in GPU memory and must be consumed by a downstream subscriber without a GPU-to-CPU copy. The custom backend presents the GPU memory as an SHM region; Zenoh routes the buffer reference rather than copying bytes.

Implementing a custom backend requires implementing the `ShmProviderBackend` trait from `zenoh::shm`.

## Buffer Lifecycle

SHM buffers are reference-counted. The buffer is released back to the provider's pool when all references are dropped, including any copy held by a subscriber. A subscriber that holds an SHM buffer reference longer than necessary depletes the provider pool, causing subsequent `alloc()` calls to block or fail.

Guidelines:
- Process or copy the payload in the subscriber and drop the sample promptly.
- Size the provider pool generously relative to the number of in-flight messages.
- Use `ZENOH_TRACE_SHM=1` with `ZENOH_LOG=trace` to observe allocation and release events during debugging.
