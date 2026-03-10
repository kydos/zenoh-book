# Getting Started

This chapter walks through installing the required tools, setting up a Rust project, and writing the first working Zenoh programs.

## Prerequisites

Zenoh 1.4.0 requires Rust stable 1.75 or later and the 2024 edition. Install Rust via `rustup`:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

After installation, verify the toolchain:

```bash
rustc --version
cargo --version
```

Zenoh's async API is built on Tokio. The `tokio` crate is a direct dependency of your application, not just of Zenoh internally — you need it in your `Cargo.toml` to use `#[tokio::main]` and to drive the async runtime.

## Installing zenohd

`zenohd` is the Zenoh router daemon. In peer mode, two processes on the same local network can discover and communicate directly without it. Once you need to cross network segments, connect machines on different subnets, or host storage nodes, you run `zenohd` as the infrastructure node.

**macOS**

```bash
brew install zenoh
```

**Ubuntu and Debian**

Add the Eclipse Zenoh apt repository, then install:

```bash
echo "deb [trusted=yes] https://download.eclipse.org/zenoh/debian-repo/ /" \
  | sudo tee /etc/apt/sources.list.d/zenoh.list
sudo apt update
sudo apt install zenoh
```

**Windows**

Download the zip archive from the [Zenoh releases page](https://github.com/eclipse-zenoh/zenoh/releases), extract it, and add the directory to your `PATH`.

**Verify the installation**

```bash
zenohd --version
```

The output shows the version string, e.g., `zenohd v1.4.0`.

## Project Setup

Create a new Cargo project:

```bash
cargo new zenoh_hello
cd zenoh_hello
```

Edit `Cargo.toml` to specify the 2024 edition and add the required dependencies:

```toml
[package]
name = "zenoh_hello"
version = "0.1.0"
edition = "2024"

[dependencies]
zenoh = "1.4.0"
tokio = { version = "1", features = ["full"] }
```

The `"full"` feature set for Tokio enables all runtime components. For production use you can trim this to the specific features you need (`rt-multi-thread`, `macros`, etc.), but `"full"` is the correct starting point.

## Hello Zenoh

The minimal Zenoh program opens a session in default peer mode and puts a single value:

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");

    let session = zenoh::open(Config::default()).await.unwrap();
    session.put("demo/hello", "Hello, Zenoh").await.unwrap();
    println!("Published.");
}
```

`zenoh::init_log_from_env_or("error")` initialises the Zenoh logger. The argument is the fallback log level when the `ZENOH_LOG` environment variable is not set. Passing `"error"` keeps output quiet during normal operation.

`Config::default()` produces a peer-mode configuration with multicast scouting enabled. No router endpoint needs to be configured.

`session.put(key, value)` is a convenience method. The key is any string that forms a valid Zenoh key (no wildcards). The value is anything that implements `Into<ZBytes>` — string literals qualify automatically.

Run the program:

```bash
cargo run
```

The output is:

```
Published.
```

The put succeeds even if no subscriber is running. Zenoh delivers to whichever subscribers exist at the time of publication; there is no error if the set is empty.

## First Pub/Sub Pair

To see data actually received, run a subscriber in one terminal before starting the publisher. Create a second binary or use `[[bin]]` sections in `Cargo.toml`. The subscriber example:

```rust
use zenoh::Config;

#[tokio::main]
async fn main() {
    zenoh::init_log_from_env_or("error");

    let session = zenoh::open(Config::default()).await.unwrap();
    let subscriber = session.declare_subscriber("demo/**").await.unwrap();

    while let Ok(sample) = subscriber.recv_async().await {
        let payload = sample
            .payload()
            .try_to_string()
            .unwrap_or_else(|e| e.to_string().into());
        println!("{}: {}", sample.key_expr(), payload);
    }
}
```

`declare_subscriber` registers interest in all keys under `demo/`. The `**` wildcard matches any number of path segments (see Chapter 3 for the full wildcard syntax).

`recv_async().await` yields the next sample from the internal channel. The loop runs until the channel is closed, which happens when the subscriber is dropped or the session closes.

`try_to_string()` attempts to interpret the payload bytes as UTF-8. It returns a `Result`; the `unwrap_or_else` branch converts decode errors to their string representation so the loop always prints something readable.

**Running the pair**

Terminal 1 — start the subscriber first so it is registered before the publisher sends:

```bash
cargo run --bin subscriber
```

Terminal 2 — run the publisher:

```bash
cargo run --bin publisher
```

Terminal 1 prints:

```
demo/hello: Hello, Zenoh
```

## Running with zenohd

In default peer mode, `zenohd` is optional. Peers on the same multicast-reachable network segment discover each other automatically via UDP multicast on `224.0.0.224:7446` and establish direct TCP connections.

To use client mode — for example, to connect two machines that cannot reach each other by multicast — start `zenohd` on one machine:

```bash
zenohd
```

Then configure each client session to connect to the router's TCP endpoint. Full configuration is covered in Chapter 4. The short form using programmatic configuration:

```rust
let mut config = Config::default();
config.insert_json5("mode", r#""client""#).unwrap();
config.insert_json5("connect/endpoints", r#"["tcp/192.168.1.10:7447"]"#).unwrap();
let session = zenoh::open(config).await.unwrap();
```

Replace `192.168.1.10` with the address of the machine running `zenohd`.

## Troubleshooting

**Publisher and subscriber do not communicate in peer mode**

Multicast scouting requires that UDP multicast traffic is allowed between the two processes. Check:

- The firewall on each machine permits UDP on `224.0.0.224:7446`.
- Both processes are on the same network segment (multicast does not cross routers by default).
- No VPN or network namespace is isolating the processes.

Set `ZENOH_LOG=debug` to see scouting messages:

```bash
ZENOH_LOG=debug cargo run --bin subscriber
```

Look for lines containing `scouting` or `scout` to confirm discovery is proceeding.

**Build fails with edition errors**

Zenoh 1.4.0 requires `edition = "2024"` in `Cargo.toml`. If you see errors about unstable features or edition-gated syntax, verify that the `[package]` section specifies `edition = "2024"` and that your Rust toolchain is 1.75 or later.

**Session opens but put/subscribe calls fail immediately**

Unwrapping on `zenoh::open` and on `put` will panic on error. During development this is acceptable; the panic message includes the underlying error. If you see a panic at session open, the most common cause is a misconfigured endpoint string — verify that the address format matches the scheme (e.g., `tcp/host:port`, not `tcp://host:port`).
