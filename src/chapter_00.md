# Zenoh Programming in Rust

This book is a practical guide to building distributed systems with [Eclipse Zenoh](https://zenoh.io) and Rust.

It targets Zenoh **1.4.0** and the Rust **2024 edition**. Each chapter introduces a concept, explains its design rationale, and demonstrates it with working code.

## Scope

| Chapter | Topic |
|---------|-------|
| 1 | What Zenoh is and where it fits |
| 2 | Setting up a Rust project with Zenoh |
| 3 | Key abstractions: keyspace, time, and data model |
| 4 | Publishing data |
| 5 | Subscribing to data |
| 6 | Queryable resources |
| 7 | Issuing queries |
| 8 | Shared-memory transport |
| 9 | Scouting and peer discovery |
| 10 | Namespaces, selectors, and QoS |
| 11 | Debugging and logging |
| 12 | Embedding Zenoh in custom systems |
| 13 | Real-world use cases: robotics, V2X, and edge-to-cloud |

## Prerequisites

- Rust toolchain installed (`rustup`)
- Familiarity with Rust ownership and async/await
- Basic understanding of pub/sub messaging

## Source Code

All examples in this book are available at <https://github.com/kydos/zenoh-book>.
